# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/task/supervised.ex
<!-- sha256: 93f06315b4d29691eabc8cfecdb6ff033c52b165f8813e66ee540c23239c893b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:46:18Z_

Both behaviors are confirmed. Let me record my final assessment.

The crash report logging `Args`/`Function` is standard OTP/`proc_lib` behavior (developer-controlled args), and `validate_stream_options` correctly rejects the dangerous values (negative timeout, non-positive concurrency). I've completed the full Phase 1 sink sweep. Here is the report.

---

## Summary

`Task.Supervised` is internal concurrency plumbing behind `Task`/`Task.async_stream`/`Task.Supervisor` where the only externally-derived data (enumerable elements) flows solely as *arguments* to developer-supplied functions and the one options-validation sink correctly rejects dangerous values; no security finding is reachable through the documented public API.

## Security

`No findings`

### Ruled out

- **`apply(module, fun, args)` in `invoke_mfa/2` (task/supervised.ex:105) — code execution.** The `module`/`fun`/`args` originate from the `Task` public API (`Task.async`, `Task.start`, `async_stream`), i.e. the developer's own callable — never from external bytes. In the `async_stream` path (task.ex:737-748 → `stream/7` → `stream_spawn/4` → `normalize_mfa_with_arg/2`, supervised.ex:507, 703-704) an enumerable *element* is prepended as the function's first **argument** (`fun.(element)`), which is data, not a callable or module name. No `String.to_atom`/`Module.concat`/`Code.eval*` on that value. Basis: verified the flow through `build_stream`/`stream_spawn` and `normalize_mfa_with_arg`.
- **`:logger.error` logging `args`/`function` in `invoke_mfa/2` (supervised.ex:116-136) — secret leakage to logs.** This is the standard OTP/`proc_lib` crash-report behavior (function + arguments in the terminating report); the arguments are whatever the developer passed to their own task, and the report is emitted via the fixed `~p`-only format in `format_report/1` (no format-string injection). Not a defect specific to this file. Basis: verified — a crashing `Task.start_link` emits `Function:`/`Args:` in the report.
- **`validate_stream_options/1` (supervised.ex:227-253) — validation sink.** Correctly rejects `max_concurrency` ≤ 0 and `timeout` < 0 / non-`:infinity`, and reaches the sink before use (task.ex:738). Residual: the `:timeout` message says "positive integer" while `0` is accepted, and `ordered`/`zip_input_on_exit` are not type-checked (used only in truthy contexts). Cosmetic/robustness, no security impact. Basis: verified — `max_concurrency: 0` and `timeout: -1` raise `ArgumentError`; `timeout: 0` and non-boolean `ordered`/`zip` are accepted with no downstream effect.
- **`Process.put(:"$ancestors"/:"$callers"/:"$initial_call")` (supervised.ex:83-93) — shared/mutable state.** These are the spawned process's own OTP ancestry/caller metadata written into its *own* process dictionary; keys are fixed and values are process lineage, not externally-controlled keys/values. By design.
- **Concurrency / races (`reply/3` `@ref_timeout`; `stream_monitor_loop`, `unlink_and_kill/1`, `handle_stop_or_parent_down/3`, inbox cleanup).** Correctness-critical, but every racer is an internal Task-machinery process (parent, monitor, task, enforcer); no external trust boundary is crossed, and the code to run is developer code. No security-relevant race reachable from untrusted input.
- **Resource consumption (`stream_reduce` spawn loop).** Concurrency is bounded by `max_concurrency` (validated > 0), and the `waiting` map is bounded by in-flight count; no unbounded growth driven by enumerable contents.
- **Other Phase-1 sink classes** — no `Code.eval*`/`compile*`/`Module.create`, no atom creation from bytes (`String.to_atom`/`binary_to_atom`/`Module.concat`), no `binary_to_term`, no `System.cmd`/`:os.cmd`/shell, no `File`/`Path`/wildcard/`:erl_tar`/`:zip`, no `EEx`, no `Base`/`URI`, no `:crypto`, no unicode/identifier handling, no `struct/2` from external maps present in this file.

## Risks / caveats

- **Scope:** the module is `@moduledoc false`; I judged it as reachable through its documented public callers (`Task`, `Task.Supervisor`). Bugs reachable only by hand-crafting internal messages to the monitor/enforcer processes, or by violating the `-spec`/header contracts (e.g. passing a non-callable MFA), are out of scope per the ground rules and were not treated as findings.
- **Standard crash-report exposure:** if a developer passes secrets as task arguments, they can appear in crash logs. This is intrinsic OTP behavior (matches `proc_lib`), controlled by the caller, and not a defect introduced here — flagged as a caveat, not a finding.
- **Not exhaustively driven:** I did not build a full adversarial `async_stream` harness exercising every timeout/kill/parent-down interleaving; the concurrency ruling rests on the trust-boundary argument (all racers are internal, developer-controlled processes) rather than on fault-injection of every schedule. The reachability of `validate_stream_options` and the crash-log content were verified by running against the in-tree Elixir (`/opt/elixir`, 1.21.0-dev).

