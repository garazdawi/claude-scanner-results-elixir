# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/task/supervised.ex
<!-- sha256: 93f06315b4d29691eabc8cfecdb6ff033c52b165f8813e66ee540c23239c893b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:55:09Z_

The file under review is byte-for-byte identical to the shipped Elixir 1.21.0-dev source, and every behavioral path I exercised (ordered/unordered streaming, timeout with `:kill_task`, `zip_input_on_exit`, option validation) behaves as documented. My review found no genuine reachable bug.

## Summary
`Task.Supervised` is byte-identical to the shipped Elixir 1.21.0-dev source and all exercised paths (ordered/unordered async_stream, timeout handling, `zip_input_on_exit`, option validation, the cross-node reply race timeout) behave correctly; I found no reachable defect.

## Bugfixes

No findings

### Ruled out

- **`validate_stream_options/1` timeout message wording** — The guard accepts `timeout >= 0` (so `0` is valid) but the `ArgumentError` reads ":timeout must be either a positive integer or :infinity". `0` (immediate timeout) is a legitimate value and the message only surfaces on the *rejection* path (e.g. `-1`), where "positive integer" is an accurate-enough description of the required class. No behavioral defect; matches upstream exactly. Basis: `verified` — `validate_stream_options(timeout: 0)` returns `%{timeout: 0, ...}`; `timeout: -1` raises the quoted message.
- **`reply/4` `after timeout` with `:infinity`** — In the `:nomonitor` branch `timeout` is `:infinity`, so `after :infinity` never fires and the task waits indefinitely for the owner's reference; this is intentional (only `:monitor` uses the 5000ms `@ref_timeout` race guard documented in the inline comment). `^mref` is `nil` there so the `{:DOWN, ^mref, ...}` clause is correctly inert. Basis: `asserted` (read of lines 21–66; `after :infinity` is standard non-firing semantics).
- **`is_reference(mref) && Process.demonitor(...)` with `nil` mref** — In the `:nomonitor` path `mref` is `nil`; `is_reference(nil)` short-circuits to `false`, so `demonitor` is not called on a non-reference. No crash. Basis: `asserted` (line 39; `&&` short-circuit is guaranteed).
- **Timer cancel on an already-fired timer (line 601–602)** — In the `:timed_out` DOWN path the timer has already fired, but `Process.cancel_timer(timer_ref, async: true, info: false)` returns `:ok` regardless of prior expiry under `info: false`, so the `:ok = ...` match holds. Basis: `asserted` (documented `cancel_timer` semantics with `info: false`).
- **`stream_cleanup_inbox/2` when parent does not trap exits** — When the parent isn't trapping exits there is no `{:EXIT, monitor_pid, _}` message queued from the link, so the `after 0` clause fires and it falls through to the `/1` flush. No block. Basis: `asserted` (lines 470–476; link exits produce no message when not trapping).
- **`get_from/1`, `get_reason/1`, `get_running/1`, `orphan_compiler_module?/1` totality** — Each has a catch-all/`else` clause (`get_reason/1` line 221, `get_from/1` line 199, `orphan_compiler_module?` `else -> false`), and `get_reason`'s `cond` handles both list-args and integer-arity stacktrace shapes for `:undef`. No unmatched shape observed. Basis: `asserted` (read of lines 142–223).
- **`stream_reduce`/`stream_deliver` accumulator threading (`max`/`spawned`/`delivered`)** — `max` is decremented on spawn (lines 400/404) and incremented when a slot frees on task-down (lines 375/379); `delivered` advances only in the delivery paths (lines 379/448); ordered delivery via `{:done, result}` and unordered via `deliver_now` both route `{:halt,_}` back to `stream_reduce/7`'s halt clause. End-to-end output ordering and counts were correct in testing. Basis: `verified` — ordered run preserved input order `[100,10,1]`; unordered run over `1..5` returned all five results.

## Risks / caveats
- The module is `@moduledoc false`; I treated its reachable surface as the `Task`/`Task.Supervisor.async_stream*` public entry points per the scope rules. Purely internal message-protocol misuse (hand-crafting the monitor/task wire messages) was out of scope and not assessed as a bug.
- I did not attempt to force the genuinely hard-to-reproduce cross-node disconnect race (lines 44–65) or the `:max_children` supervisor-exhaustion path under live conditions; those were reviewed by reading only and their guards appear internally consistent.
- Confirming byte-identity to `/opt/elixir` (1.21.0-dev) rules out a *planted* divergence from that build but does not by itself prove the upstream logic bug-free; the ruled-out entries above rest on the behavioral tests and reasoning cited, not on the identity alone.

