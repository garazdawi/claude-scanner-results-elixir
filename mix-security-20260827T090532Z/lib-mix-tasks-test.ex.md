# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/test.ex
<!-- sha256: 57b020574f4d00607063d1e81dfda0db72ced3fa1231feb57b1d59910c358ac8 -->

_Analysed on 2026-08-27T11:28:24Z_

Confirmed all three behaviors. Let me record what matters:

- `Module.concat(["..."])` **mints new atoms** (`new_atoms=2`) — the atom-injection sink shape is real, but its input is the `--formatter` CLI value.
- `Path.wildcard` compiles the glob from config/CLI-supplied `path`/`pattern`.
- The partition parse is robust: `"3junk"` → `{3,"junk"}` fails the `{partition, ""}` match, `"-1"`/`"0"`/oversized all fail the `1..total//1` range guard, `""` → `:error` — every hostile `MIX_TEST_PARTITION` value hits a clean `Mix.raise`, no crash and no injection.

Every sink in this file traces to developer CLI args, operator `mix.exs`/config, operator env vars, or the project's own `test_helper.exs`/test files — none to a network peer, archive, git ref, dependency, or untrusted read-file. That places them outside this codebase's trust boundary.

## Summary

`Mix.Tasks.Test` (the `mix test` task) contains no security vulnerability reachable across a trust boundary: every sink (atom-minting `Module.concat`, `Path.wildcard`, `Code.require_file`, coverage-tool dispatch, partition parsing, umbrella path math) is fed only by developer-typed CLI arguments, operator-controlled `mix.exs`/config/env, or the project's own test files, all of which are out of scope by design.

## Security

No findings

### Ruled out

- **`Module.concat([&1])` on the `--formatter` value (`formatter_opts/1`, line 998)** — this is the classic atom-injection sink and I verified it mints brand-new atoms (`Module.concat(["Totally.Bogus...N"])` created 2 atoms, uncollectible per CWE-770). But the input is `Keyword.get_values(opts, :formatter)`, sourced solely from the `--formatter` command-line switch typed by the developer running `mix test`. No dependency, archive, network peer, or read-file influences it, so it does not cross a trust boundary. (`Module.safe_concat/1` would still be the hardening if one wanted defense-in-depth against a wrapper script passing untrusted formatter names.)
- **`Path.wildcard("#{path}/**/#{pattern}")` (`extract_files/2`, line 777)** — cost/pattern are caller-controlled, but `path` comes from CLI file args or `project[:test_paths]` and `pattern` from `project[:test_pattern]`; both are developer/operator inputs. No hostile source.
- **`Code.require_file(file)` for `test_helper.exs` (`require_test_helper/1`, line 1092)** — executes code, but the file is the project's own `test/test_helper.exs`, documented behavior and developer-authored. Not attacker-controlled.
- **`cover[:tool].start(compile_path, cover)` (`do_run/3`, line 625)** — module-name dispatch, but the module comes from `project[:test_coverage][:tool]` (operator `mix.exs`) or the hard-coded default. Operator config is out of scope (already RCE by design).
- **`filter_by_partition/3` + `Integer.parse(MIX_TEST_PARTITION)` (lines 1040–1072)** — I ran the hostile inputs: `"3junk"`, `"-1"`, `"0"`, a 24-digit integer, and `""` all fail either the `{partition, ""}` exact match or the `1..total//1` range guard and produce a clean `Mix.raise`. No crash, no path/option injection; and the env var is CI/operator-controlled regardless.
- **Umbrella path handling `Path.expand`/`Path.relative_to`/`Path.join("../..", file)`/`String.trim_leading` (lines 541–570) and `relative_app_file_exists?`** — all components derive from CLI file arguments and `Mix.Project.config()[:app]` (mix.exs); the only filesystem touch is `File.exists?` (a read/probe, no write/delete/link), so even a `..`-laden argument only affects which of the developer's own paths is stat-ed. The `{[file], _} = parse_paths([file])` match is safe because `parse_paths` maps 1:1 over its input.
- **`get_manifest_path/1` → manifest read/write (line 1008) and `--failed` (`manifest_opts/1`)** — the manifest path comes from `--failures-manifest-path`/app-env/default under `_build`; the manifest is written by `mix test` itself, not an untrusted producer. `ExUnit.Filters.failure_info` parsing lives in another file (out of review scope).
- **`--listen-on-stdin` → `System.restart()` (lines 592–599)** — triggered by a developer opt plus a newline on stdin from a local file-watcher; no remote boundary.
- **`--name-pattern`/`:regex` and `--include`/`--exclude`/`--only` filters** — potential ReDoS/atom sources, but the regex and filter strings come from CLI switches (developer), and the atom conversion (`String.to_unsafe_atom` in `ExUnit.Filters.parse`) happens in another file.

## Risks / caveats

- **Adjacent files not reviewed.** The actual require/compile/run of test files (`Mix.Compilers.Test.require_and_run`), the manifest reader (`ExUnit.FailuresManifest`/`ExUnit.Filters` — note `parse/1` uses `String.to_unsafe_atom` on filter keys), and the coverage tool (`Mix.Tasks.Test.Coverage`, which consumes the CLI/env-derived `:export` filename as a write path) are separate files and out of scope here; a path-traversal or atom concern, if any, would live there, not in `test.ex`.
- **Trust-model dependent.** The conclusion rests on the folder threat model: `mix.exs`/config and the developer's CLI are trusted (they already imply RCE). If someone runs `mix test` on a fully untrusted checkout, the untrusted `mix.exs` is the RCE vector long before any sink in this file — so nothing here adds marginal risk under that scenario either.
- **Verified vs. asserted.** I ran the atom-minting, wildcard, and partition-parse probes against the built source (identical to the file under review, Elixir 1.21.0-dev/OTP 29). I did not stand up a full umbrella project to exercise the recursive path branch end-to-end; that path was analyzed statically and reasoned about, not executed.

