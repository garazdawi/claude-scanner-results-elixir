# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.ex
<!-- sha256: 095e1308ebecec677b9c813a8b83e75bb8b4c11a883ff3e12d28de3d0334db26 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.ex
<!-- sha256: 095e1308ebecec677b9c813a8b83e75bb8b4c11a883ff3e12d28de3d0334db26 -->

_Re-reviewed on 2026-08-27_

The two first-pass hedges ("I assumed") both concerned neighbour-module return shapes. I read `project.ex:910` (`consolidation_path/1`) and `task.ex:216`/`task.ex:297` (`moduledoc/1`, `task_name/1`) and confirmed both. Neither hedge became a finding; both are now verified from source. The single security finding is unchanged and retains its executed evidence.

## Summary

`Mix.Tasks.Compile` is a thin orchestrator whose only externally-derived sink is `load_protocol/1`, which interns atoms and purges/deletes modules from consolidation-directory filenames via the explicitly-unsafe `String.to_unsafe_atom/1`; the `--erl-config`, `--list`, and diagnostics paths are operator- or trusted-metadata-controlled.

## Security

#### `CSCAN-compile-consolidation-atom-pollution`
- **Finding ID** — `CSCAN-compile-consolidation-atom-pollution`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `lib/mix/lib/mix/tasks/compile.ex:225` (`load_protocol/1`), reached from `run/1` at line 184–188.
- **Class** — Atom creation (CWE-770); secondary: unguarded crash / forced code unload.
- **Impact** — `load_protocol/1` takes each filename returned by `File.ls(Mix.Project.consolidation_path())`, and for any entry beginning with `"Elixir."` runs `file |> Path.rootname() |> String.to_unsafe_atom()` and then `:code.purge/1` + `:code.delete/1` on the result. `String.to_unsafe_atom/1` is `:erlang.binary_to_atom(s, :utf8)`; its own docs warn it "should not be an untrusted value … Consider using `to_existing_atom/1` instead." Any entity that can write files into the consolidation directory (`_build/<env>/lib/<app>/consolidated` or, for umbrellas, `_build/<env>/consolidated`) — e.g. a poisoned or restored CI `_build` cache, a shared umbrella build directory — can therefore, on the next incremental `mix compile`: (a) intern one permanent, non-GC'd atom per stray `Elixir.*` file, pushing toward the 1,048,576-atom ceiling and eventual VM abort; (b) force `:code.purge`/`:code.delete` of any *real* loaded module whose name they choose as a filename (`Elixir.MyApp.Repo.beam` → `MyApp.Repo` purged/deleted mid-build, killing processes lingering on that module's code); (c) crash the whole `compile` run with an uncaught `ArgumentError` by planting a filename containing invalid UTF-8. The incremental compile path does not force-clean the directory (`loadpaths.ex:70` wipes it only on toolchain/SCM version change; `protocol.ex`'s `diff_manifest` removes only specific known protocol beams), so stray files persist to be processed. Reachability requires write access to the build tree, which is why this is Low rather than higher.
- **Evidence** — `verified`. `String.to_unsafe_atom/1` = `:erlang.binary_to_atom(string, :utf8)` (installed `string.ex:3040`). Replaying the exact `load_protocol/1` body over 2000 hostile filenames: `brand-new atoms interned from 2000 stray filenames: 2029`, each result `{:interned, <atom>}` becoming a purge/delete target. Invalid-UTF-8 filename `"Elixir." <> <<0xFF,0xFE>> <> ".beam"`: `CRASH: {ArgumentError, "…1st argument: invalid UTF8 encoding…"}` — unguarded in `load_protocol/1`, so it propagates out of `run/1`. The sink directory is confirmed to be `_build/.../consolidated`: `Mix.Project.consolidation_path/1` (project.ex:910–917) returns `config[:consolidation_path] || Path.join(build_path|app_path, "consolidated")` — no lower-trust channel selects it; the override key is operator project config.
- **Suggested fix** — Use `String.to_existing_atom/1` and skip on failure: a module that was never loaded has nothing to purge or delete, so `rescue ArgumentError -> :ok` (or `try/1`) both removes the atom-pollution/crash sink and preserves behavior for the legitimate consolidated beams (whose module atoms already exist). Equivalently, restrict processing to entries that resolve to already-loaded modules.

### Ruled out

- **`load_erl_config/1` — `:file.consult` + `Application.put_all_env(persistent: true)` (lines 212–217).** `verified`. The path comes only from the documented `--erl-config` CLI switch parsed at line 141 (`switches: [erl_config: :string]`), i.e. the operator; there is no dependency/network/archive channel that injects it. `:file.consult` parses Erlang terms and does not evaluate code, so no RCE. Setting global app env from an operator-chosen config file is the documented purpose. (The `{:ok, terms} = …` hard match raising `MatchError` on a bad operator path is a robustness nit, not a security issue.)

- **`--list` output: `format/2` / `first_line/1` over `Mix.Task.moduledoc/1` (lines 110–127).** `verified`. I read `Mix.Task.moduledoc/1` (task.ex:216–222): it returns the moduledoc string from `Code.fetch_docs(module)` (or `nil`/`false`), and `Mix.Task.task_name/1` (task.ex:297–299) delegates to `Mix.Utils.module_name_to_command/2` — both produce plain strings that flow into `format/2` as `~ts`/`~-Ns` *arguments*, never as the `:io_lib.format` control string (the literal `~c"mix ~-#{max}s # ~ts"`, where `max` is an integer from `byte_size`). `~ts`/`~s` do not re-interpret `~` directives contained in their argument, so a moduledoc originating from an installed archive's `compile.*` task cannot inject format directives. Any terminal-escape content is strictly weaker than the arbitrary-code execution an installed archive already has by design (archives load as plugins, "install only from sources you trust"), so it adds no capability.

- **`Code.prepend_path/1`, `Code.prepend_paths/1`, umbrella `apps_paths` (lines 158–188).** `verified`. All paths derive from `Mix.Project.config()` / umbrella layout (`apps_paths`, `consolidation_path` — confirmed operator-config-derived at project.ex:910–917), not from a lower-trust source.

## Did I verify this?

- **Consolidation-path location (first-pass hedge "I assumed `consolidation_path` returns …"):** RESOLVED — verified by reading `project.ex:910–917`. It returns `_build/<env>/lib/<app>/consolidated`, the umbrella `build_path/consolidated`, or an operator-supplied `config[:consolidation_path]`. No lower-trust injection of the path; the finding's reachability premise (an attacker who can *write files into* that directory) is unchanged.
- **`moduledoc`/`task_name` transitive shapes (first-pass hedge "I assumed …"):** RESOLVED — verified by reading `task.ex:216–222` and `task.ex:297–299`. Both return plain strings consumed as format *arguments*, confirming the `--list` ruled-out entry. No hedge remains.
- **Sink behaviors (atom interning + invalid-UTF-8 crash):** verified in the prior pass by faithfully replaying the `load_protocol/1` body in the built OTP-29 toolchain; evidence retained above.
- **Remaining honest limitation (not a hedge, a scope note):** I did not run a full end-to-end `mix compile` against a physically poisoned consolidation directory, so the last hop — a planted file surviving a real incremental build and reaching `load_protocol` — is established by source tracing (`loadpaths.ex`/`protocol.ex` do not force-clean stray files on an incremental build) rather than an executed end-to-end repro. This is why the finding is Confidence **medium**, Severity **Low**, and gated on build-tree write access.

## Dropped in re-review

None. No prior finding was dropped; both first-pass "I assumed" hedges were upgraded to verified via source reading (neither warranted a new finding), and the single security finding is preserved with its executed evidence.
