# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/release.ex
<!-- sha256: 92ef5d0df8013ae819f26b68b506cd2e56645c5f9f35a52f6739523913d238b2 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/release.ex
<!-- sha256: 92ef5d0df8013ae819f26b68b506cd2e56645c5f9f35a52f6739523913d238b2 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

## Summary

`lib/mix/tasks/release.ex` is the `mix release` task: a thin orchestration layer over `Mix.Release`, `Mix.Generator`, and `Mix.Tasks.Release.Init`. Its only documented public entry point is `run/1`; the rest is private glue that copies files, renders templates, and builds the tar. I found no genuine, reachable bugs — the control flow, error propagation, and template selection all hold up under inspection and targeted runs. In this pass I read the full `Mix.Tasks.Release.Init` template source (all five templates embedded via `embed_template/2`) and executed the `vm_args_text/1` divergence, retiring the "did not exhaustively render every template" hedge.

## Bugfixes

No findings.

## Risks / caveats

- The bulk of the real work is delegated to `Mix.Release.*` (boot scripts, sys.config, cookie, ERTS/app copying, tar helpers) which live in **other files** (`lib/mix/release.ex`) and are reviewed under those files. This is a scope boundary, not an unverified assumption about this file: every `Mix.Release.*` call site here consumes the documented `:ok | {:error, message}` / struct return shapes, and the `with` chain in `build_rel/2` (lines 1230–1242) correctly funnels any `{:error, message}` to `File.rm_rf!(version_path)` + `Mix.raise/1`. A defect *inside* those callees would surface in their own review, not here. Confidence: not applicable (scope demarcation).

- Behaviour of the generated artifacts depends on EEx templates embedded from `Mix.Tasks.Release.Init`. In this pass I read all five (`vm_args_text/1`, `env_text/0`, `cli_text/0`, `env_bat_text/0`, `cli_bat_text/0`) at `release.init.ex:46–513`. The templates reference only `@release.name` (an atom, rendered via EEx `to_string`) and the same-module private `release_mode/2` (in scope for the compiled template functions since `embed_template` compiles them into `Mix.Tasks.Release`). No missing assign or out-of-scope reference. Confidence: not applicable (verified, no defect).

- Reachability was judged against the documented public API (`run/1` and the `:steps` contract). Findings requiring callers to violate the documented `@doc`/`@spec` (e.g. a step function of the wrong arity, a malformed `mix.exs` `:releases` config) are out of scope by the given rules.

### Ruled out

- **`String.to_unsafe_atom(name)` on the CLI release name (line 1071) — not a bug.** The threat model flags atom creation from external strings, but the input is operator-controlled CLI argv, the function exists and works, and unbounded atom growth from a single operator-supplied name is not a reachable DoS. Intended behaviour, matching every other Mix task-name path.
  Axes examined: input-provenance (CLI argv, operator-controlled), function-existence/return (`elixir -e 'IO.inspect String.to_unsafe_atom("foo")'` → `:foo`, executed) | Not examined: none.
  Confidence: high (not a finding).

- **`make_boot_scripts/3` returning via `Enum.find(results, :ok, &(&1 != :ok))` (line 1320) — correct.** `Enum.find/3` returns the first non-`:ok` element (surfacing the first error to the `with` in `build_rel`) and defaults to `:ok` when all succeed. Executed: `elixir -e 'Enum.find([:ok,:ok,{:error,1},:ok], :ok, &(&1 != :ok))'` → `{:error, 1}`; all-`:ok` → `:ok`. Each `for` iteration correctly ends in `:ok` after the rename/rm side effect.
  Axes examined: arg-order/default semantics (executed both mixed and all-`:ok` inputs), per-iteration return value (each branch of the `if` at 1309–1316 ends in `:ok`) | Not examined: none.
  Confidence: high (not a finding).

- **Default `remote.vm.args` reusing the `:vm_args` (`vm_args_text(false)`) template when `rel/remote.vm.args.eex` is absent (lines 1323–1333, embed at line 1510) — not a functional bug.** `make_vm_args/2` writes `vm_args_template(release: release)` — i.e. `vm_args_text(false)` — for **both** `vm.args` and `remote.vm.args` when no `.eex` override exists. Executed a byte-for-byte comparison of `vm_args_text(false)` vs `vm_args_text(true)`: they differ on exactly one line — `##-start_epmd false -erl_epmd_port 6789` vs `##-start_epmd false -erl_epmd_port 6789 -dist_listen false` — and **both variants of that line begin with `##` (commented out)**, so the generated file is functionally identical either way. A user wanting epmd-less deployment edits/generates the `.eex` (via `mix release.init`, which does emit `vm_args_text(true)` for `remote.vm.args.eex`), so the divergence never reaches a live flag.
  Axes examined: text-equality of the two variants (executed: `false_text == true_text` → the single differing line identified), comment-state of the differing line in both variants (executed: both `starts_with "##"` → true) | Not examined: none.
  Confidence: high (not a finding). *(Hedge "did not exhaustively render every template" from the first pass is now retired — all five templates read and the vm_args divergence executed.)*

- **`OptionParser.parse!(..., strict: @switches)` (line 1069) rejecting unknown switches — matches contract.** All switches documented under "Command line options" are present in `@switches`; a strict parse raising on an undocumented flag is the documented behaviour, not a defect.
  Axes examined: switch-list vs documented options (cross-checked `@switches` lines 1043–1053 against the "Command line options" list lines 1029–1037) | Not examined: none.
  Confidence: high (not a finding).

- **`maybe_add_config_reader_provider/3` `cond` fall-through (lines 1277–1293) — total.** The three-way `path` / `config_providers == [] and path != false` / `true` branches cover string-path, no-config-found, and has-providers-or-`false` cases; each returns a `Mix.Release` struct. No shape is left unhandled.
  Axes examined: branch exhaustiveness over `{path, config_providers}` states (string path, `nil`, `false` × empty/non-empty providers), each branch's return type (all yield the struct) | Not examined: none.
  Confidence: high (not a finding).

- **`copy/2` for `:executables` writing `env.sh`/`env.bat` and CLIs (lines 1413–1447) — distinct filenames per OS, no clobber.** `cli_for(:unix, …)` yields `"env.sh"` + `"#{name}"`; `cli_for(:windows, …)` yields `"env.bat"` + `"#{name}.bat"`; `elixir_cli_for/2` yields `elixir`/`iex` vs `elixir.bat`/`iex.bat`. Unix and windows passes write different targets, so a `[:unix, :windows]` release does not overwrite.
  Axes examined: filename collision across the two OS passes (compared `cli_for/2` and `elixir_cli_for/2` outputs — env script, launcher, and the two elixir CLIs all differ by `.bat` suffix) | Not examined: none.
  Confidence: high (not a finding).

## Did I verify this?

Yes, for every claim carried in this pass:

- Read the full target file (`release.ex`, 1516 lines) and the full neighbour `release.init.ex` (515 lines), covering all five embedded templates.
- Executed the `vm_args_text(false)` vs `vm_args_text(true)` comparison in a working Elixir toolchain: confirmed they differ on a single commented-out (`##`) line, so the default `remote.vm.args` is functionally identical to `vm.args` — the sole first-pass hedge ("did not exhaustively render every template") is discharged with executed evidence.
- The `Enum.find/3`, `String.to_unsafe_atom/1`, `OptionParser` strict-parse, `cond`-totality, and per-OS-filename ruled-out entries were verified by execution and/or direct cross-reference; each now carries `Not examined: none`.
- The `Mix.Release.*` delegation remains a genuine other-file scope boundary (reviewed under `lib/mix/release.ex`), not an unverified assumption about this file's own logic — I did not claim to verify those callees' internals and did not upgrade them to findings, because nothing observable in this file contradicts their documented contracts.

### Dropped in re-review

No findings were dropped (the first pass reported none). No ruled-out entry was downgraded or removed; one hedge (template rendering) was upgraded to `verified` with executed evidence as described above.
