# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/release.ex
<!-- sha256: 92ef5d0df8013ae819f26b68b506cd2e56645c5f9f35a52f6739523913d238b2 -->

_Analysed on 2026-08-27T11:25:24Z_

I have completed a thorough trace of every sink in this file. Let me summarize my methodology before writing the report:

- **Sink inventory**: The file contains no `System.cmd`/`:os.cmd`/`System.shell`, no `Code.eval*`/`compile*`, no `binary_to_term`, no `Regex.compile`, no network calls. Its only sinks are `Path.join` + `File.*` operations, `:erl_tar.create`, EEx `copy_template`, and one atom creation.
- **Origin of every path component**: all operator-controlled (release config, `rel/` templates, project config paths) **except** application `name`/`vsn` (line 1175) which come from dependency `.app` metadata.
- **Verified behaviorally**: `String.to_unsafe_atom/1` is the real Elixir 1.21 API (input is CLI argv); `Path.join` and `:erl_tar.create` do not sanitize `..`; `Mix.Release` version validation is empty-string-only.
- **Key trust reasoning**: the real write-sinks for a hostile `vsn` (`mkdir_p!`/`cp_r!` on `lib/#{app}-#{vsn}`) and the version-validation gap both live in `Mix.Release` (another file); and any malicious dependency has already achieved RCE during the mandatory `mix compile` (line 1062) that precedes assembly.

## Summary

`Mix.Tasks.Release` orchestrates `mix release` over operator-controlled inputs (project config, `rel/` templates, release options); its only untrusted input is dependency `.app` metadata, whose dangerous sinks (path construction, validation) live in sibling `Mix.Release` and are moot given that any hostile dependency already ran code during the mandatory pre-assembly `mix compile` — no security defect is present in this file itself.

## Security

No findings.

### Ruled out

- **`String.to_unsafe_atom(name)` — atom-table exhaustion (line 1071).** `name` is the CLI argument to `mix release NAME`, i.e. operator-typed argv, which is outside the attacker model (the operator is not the attacker). Verified `String.to_unsafe_atom/1` is the genuine Elixir 1.21 API — `function_exported?(String, :to_unsafe_atom, 1)` is `true`, its docs describe it as "Converts a string to an existing atom or creates a new one" — semantically identical to the historic `String.to_atom` used upstream; this is not an injected weakening. No dependency/network/file boundary can inject `name`.
- **`File.rm_rf!(version_path)` / tar `out_path` with traversal-bearing `release.version` (lines 1169, 1208, 1239).** `release.version` can be pulled from a dependency's raw `vsn` string via the documented `version: {:from_app, app_name}` feature, and I verified both `Path.join` and `:erl_tar.create` preserve `..` unmodified (`Path.join(["…/releases","1/../../../../tmp/x"]) => "…/releases/1/../../../../tmp/x"`). However, the missing validation is in `Mix.Release.from_config!` (other file), which checks only for the empty string (release.ex:159-163) before building `version_path`; `build_rel` here is entitled to treat the `Mix.Release` struct's `version_path` as well-formed per its documented contract. The defect and its fix belong to `Mix.Release`, not this file.
- **`make_tar` — dependency `name`/`vsn` become tar entry names (lines 1175, 1200).** A hostile `vsn` yields a `lib/app-…/../…` entry, and `:erl_tar.create` does not sanitize it (verified: entry `~c"lib/app-1/../../../evil"` survives into the archive). But the entry names merely mirror on-disk directories that `Mix.Release.copy_app` already created (`mkdir_p!` on the same `lib/#{app}-#{vsn}`), so `make_tar` introduces no traversal beyond what the other file already performed on disk; the `File.exists?` filter (line 1195) further requires the source to exist. Not an independent sink here.
- **`:erl_tar.create(..., [:dereference, ...])` (line 1200).** `:dereference` bundles symlink targets into the operator's own release tarball; app dirs were already copied with `dereference_symlinks: true` by `copy_app`, so this re-materializes already-real files. The artifact belongs to the operator and no privilege boundary is crossed at creation time.
- **`Config.Reader.read!(config[:config_path], …)` (line 1219) executing config as code.** `config[:config_path]` is the project's own `config/config.exs` (operator). Executing project config at build time is by design, equivalent to `mix.exs` RCE — explicitly out of scope.
- **EEx `copy_template` for `vm.args`/`env.sh`/`env.bat` (lines 1327, 1424) and function `:steps` (line 1091).** Template files come from `rel_templates_path` = `release.options[:rel_templates_path] || "rel"` (operator's project dir, confirmed), and steps are anonymous functions from the operator's `mix.exs`. Both are operator-authored code executing on the operator's build machine — RCE by design, not a boundary crossing.
- **`File.cp_r!` of overlays (line 1385).** Overlay directories come from `release.options[:overlays]` and `rel/overlays` — operator configuration.
- **`Task.async_stream` copy fan-out (line 1153) / overwrite `File.exists?` prompt (line 1080).** No external racer exists on the operator's build directory; the only interleavable actor would be another local process, which is not in this threat model.

## Risks / caveats

- **Cross-file validation gap (not this file):** `Mix.Release.from_config!` does not validate that `:version` is a well-formed version string (only rejects `""`), so a dependency's arbitrary `vsn` — reachable through the documented `version: {:from_app, dep}` option, which the docs endorse for umbrellas — flows unsanitized into `version_path` and into `lib/#{app}-#{vsn}` used by `copy_app`'s `mkdir_p!`/`rm_rf!`/`cp_r!`. If reviewing `Mix.Release`, that missing sanitization (and the `File.rm_rf!(version_path)` consumer at release.ex:1208) is the place to add a `Version`/path-component check. I did not report it here because both the unchecked value and its primary write-sinks originate in `Mix.Release`.
- **Moot-by-RCE assumption:** my dismissal of dependency-metadata attacks rests on the fact that any dependency is compiled (its code runs) before `mix release` assembles. A pathological, never-compiled precompiled dependency whose modules are never invoked could in principle deliver a hostile `vsn` without prior RCE; that narrow case would make the cross-file traversal a genuine (Low) escalation, but the fix still lives in `Mix.Release`, not this file.
- **Not executed end-to-end:** I did not drive a full `mix release` with a crafted dependency to observe a traversal write, because the destructive/creating sinks (`mkdir_p!`, `cp_r!`) are in the out-of-scope sibling file; I verified the enabling primitives (`Path.join`/`:erl_tar.create` non-sanitization, `String.to_unsafe_atom` identity, `from_config!` version check) directly and reasoned about the rest.

