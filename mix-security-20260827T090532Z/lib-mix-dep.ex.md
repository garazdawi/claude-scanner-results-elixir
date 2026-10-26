# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep.ex
<!-- sha256: faca7f2a34792c70c4eb1d0e7a0a87effdc1d72ce7e2dbe27601ee09128f59ed -->

_Analysed on 2026-08-27T09:34:20Z_

Verified end-to-end. `format_status/1` output flows to `shell.error(...)`/`shell.info(...)` (fetcher.ex:176, deps.loadpaths.ex:201, deps.ex:233) — printed to the developer's terminal during `mix deps`, `mix deps.compile`, `mix deps.get`, and `mix compile`. I have enough to finalize.

## Summary

`Mix.Dep` is dependency-metadata glue whose sinks (atom creation, `:application.load`, SCM/behaviour dispatch, project entry) are almost all either operator/CLI-trusted or by-design dependency-code execution; the one dependency-controlled-data defect is an inconsistent-escaping terminal-injection in `format_status/1`.

## Security

#### `CSCAN-dep-noappfile-escape-injection`
- **Finding ID** — `CSCAN-dep-noappfile-escape-injection`
- **Severity** — Low
- **Confidence** — high
- **Location** — `elixir/lib/mix/lib/mix/dep.ex:304` (and `:299`), `format_status/1` `{:noappfile, {path, other_path}}` clause
- **Class** — Template / interpolation (terminal/log line, CWE-150/117)
- **Impact** — `other_path` is the `.app` filename discovered by `Path.wildcard(".../ebin/*.app")` (loader.ex:484) inside a dependency's build directory, i.e. a name the dependency controls. Line 299 takes its basename into `other_app` and line 304 interpolates it **raw** (`try changing the dependency name to :#{other_app}`), while the very next reference on line 303 wraps the same path in `inspect(...)`, which neutralizes control bytes. A dependency whose app file is named with ANSI/terminal control sequences (e.g. `ebin/\e[2K\e[1G... .app`) therefore injects raw escape bytes into the dependency-status line Mix prints via `Mix.shell()` (`shell.error`/`shell.info` at fetcher.ex:176, deps.loadpaths.ex:201, deps.ex:233) during `mix deps`, `mix deps.compile`, `mix deps.get`, or `mix compile`. Consequence is terminal output spoofing — hiding the real error, faking an "ok"/different-dependency line, or repositioning the cursor to mislead the developer. Escalation beyond a dependency's existing build-time capabilities is marginal (hence Low), but it is dependency-sourced data reaching an un-escaped sink where the sibling value is escaped.
- **Evidence** — `verified`. Constructed a `%Mix.Dep{status: {:noappfile, {ok_path, "…/ebin/\e[31mFAKE-Mix-error\e[0m.app"}}}` and called `Mix.Dep.format_status/1`:
  ```
  $ elixir repro_status.exs
  === does the RAW ESC byte (27) survive into the message? ===
  contains ESC byte 27: true
  count of ESC bytes in message: 2
  ```
  Inspecting the returned message shows the `inspect`-wrapped `other_path` rendered safely as literal `\e[31m…` text (zero raw ESC bytes), while the two raw ESC bytes (27) present in the message come entirely from the raw `:#{other_app}` interpolation.
- **Suggested fix** — Escape `other_app` consistently with the adjacent path, e.g. `try changing the dependency name to :#{inspect(other_app)}` (yields `:"…"` with control bytes escaped), or strip non-printable characters before interpolating into shell output.

### Ruled out
- **Atom creation via `String.to_unsafe_atom/1` in `to_app_names/1` (line 558, reached from public `filter_by_name/3`).** `String.to_unsafe_atom/1` is `:erlang.binary_to_atom/2` (confirmed in string.ex:3040) and the atom table is never GC'd, but the binary branch is only ever fed developer-typed CLI app names: `deps.compile.ex:88` (`tail`), `dep/fetcher.ex:34` (`by_name` CLI `names`). Every programmatic caller passes atoms (`deps.loadpaths.ex:181` maps `& &1.app`; `fetcher.ex:100` passes the converger's atom `apps`), so `is_binary(app)` is false and no atom is created. No dependency/network/lockfile route supplies binaries here; the operator-CLI case is explicitly out of scope, and a dependency that could feed strings already has build-time RCE, so this is not an escalation.
- **`:application.load({:application, app, app_properties})` in `converge_and_load/1` (line 168).** `app_properties` come from the dependency's own `.app` file, but `:application.load/1` only registers the resource and namespaces `env` under the dep's own `app` atom; it does not start the app or run its `mod` callback. Consuming a dependency's app resource is the intended function of Mix and confers no cross-app poisoning.
- **`Mix.Project.in_project(app, opts[:dest], config, fun)` in `in_dependency/3` (line 279).** Entering `opts[:dest]` and evaluating the dependency's `mix.exs` is by-design build-time execution of dependency code (documented threat-model out-of-scope: the operator chose the dependency). `opts[:dest]`/`opts[:build]` are Mix-computed, not externally injected paths.
- **Behaviour dispatch `scm.lock_status/1`, `scm.fetchable?/0`, `scm.format/1` (lines 263, 398, 450, 512).** `dep.scm` is resolved to a registered `Mix.SCM` module during dep resolution, not an attacker-chosen arbitrary module; no untrusted string is turned into a dispatch target here.
- **`get_children/3` + `populate_seen/2` recursion (lines 226-240).** Termination and cost are bounded by the distinct-app set via the accumulating `seen` MapSet (O(1) membership); a large graph is bounded by declared deps and cannot force super-linear blowup.

## Risks / caveats
- The terminal-injection trigger requires the dependency to reach the `{:noappfile, {path, other_path}}` state (declared app name differs from the compiled `.app` basename found in its ebin). I verified byte survival and the print sinks, but did not stage a full malicious-dependency `mix deps` run end-to-end; the message-construction and shell-output paths are confirmed by source and unit-test fixtures (deps_test.exs:189-201).
- I treated `String.to_unsafe_atom` per its installed definition (Elixir 1.21.0-dev, `binary_to_atom/:utf8`); the full Elixir stdlib is not in this checkout (only `lib/mix`), so cross-module claims about callers rely on the mix subtree that is present.
- Whether a dependency can populate its build `ebin` with an arbitrarily-named `.app` file without executing build code (e.g. certain path/prebuilt-artifact layouts) was not exhaustively enumerated; it does not change the finding's validity, only how much it adds over a dependency's existing capabilities.

