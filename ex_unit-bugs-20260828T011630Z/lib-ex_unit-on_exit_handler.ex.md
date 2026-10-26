# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/on_exit_handler.ex
<!-- sha256: e39b79916a4e46c53a1c83e6dd497912cf785ac0c03260d7f571d400495d8e8a -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:35:51Z_

All three copies (the review file, the built `/opt/elixir`, and `/cscan/0`) are byte-identical, so the file matches the shipped Elixir 1.21-dev ExUnit. I ran the real `start_supervised` teardown path to confirm the supervisor-termination design (it relies on the process link, not an explicit `Process.exit`). Here is my assessment.

## Summary

The file is a faithful copy of the shipped ExUnit `OnExitHandler` and is behaviorally sound on all normal paths; the only defensible defect is a missing `Process.demonitor/2` flush on the rare supervisor-shutdown-timeout path, which leaks a stray `:DOWN` message.

## Bugfixes

#### `CSCAN-on_exit_handler-supervisor-timeout-monitor-leak`
- **Finding ID:** `CSCAN-on_exit_handler-supervisor-timeout-monitor-leak`
- **Severity:** Low
- **Confidence:** low
- **Location:** `on_exit_handler.ex:61-70`, `terminate_supervisor/2` (timeout branch)
- **Class:** Process and message errors (monitor never cleaned up)
- **Impact:** When the per-test supervisor fails to shut down within `timeout` after the test process exits, `terminate_supervisor/2` returns the `TimeoutError` tuple but leaves the monitor `ref` active. If the supervisor dies later, a `{:DOWN, ^ref, …}` message is delivered into the ExUnit `Runner` process mailbox and is never consumed. It cannot match any subsequent selective `receive` (those use fresh refs), so it lingers harmlessly until that process exits — no test result is corrupted, but it is an untidy monitor/message leak on the error path. Contrast the success branch (`{:DOWN, ^ref, …} -> nil`), which consumes the DOWN and auto-removes the monitor, and the analogous `receive_runner_reply/4` kill path, which explicitly drains the DOWN.
- **Evidence:** asserted. The timeout branch (lines 66-69) returns the error tuple with no `Process.demonitor(ref, [:flush])` and no drain `receive`; the monitor established at line 62 therefore stays live. Triggering it requires a supervised child that ignores `:shutdown` long enough to exceed the configured `:timeout`, which I did not construct into a clean observation. I verified the normal path runs and cleans up in ~6 ms via a `start_supervised` test (`elixir sup_test.exs` → `1 passed`, `ELAPSED_MS=6`).
- **Suggested fix:** In the `after timeout ->` branch, call `Process.demonitor(ref, [:flush])` before returning the error tuple.

## Risks / caveats
- The file is **byte-identical** to `/opt/elixir/.../on_exit_handler.ex` (the interpreter actually running) and to `/cscan/0/.../on_exit_handler.ex`; there is no pristine upstream in the sandbox and no network, so I could not diff against the canonical Elixir repo to confirm whether the flush omission above is a genuine deviation or matches shipped behavior. I therefore rated it Low/low.
- The one finding lives on a rare error path of an `@moduledoc false` internal module reached only through the `Runner`, not through a documented public API; per the scope rules it is borderline, but I surfaced it rather than silently ruling it out.

### Ruled out
- **`terminate_supervisor/2` never calls `Process.exit(sup, …)`** — not a bug. The test supervisor is started with `Supervisor.start_link` *from inside the test process* (`ex_unit.ex:549` via `fetch_test_supervisor`), so it is **linked** to the test process. The test process exits `:shutdown` after sending `:test_finished` (`runner.ex:466-467`), which shuts the linked supervisor down; `terminate_supervisor` only needs to monitor and await the DOWN. Verified: a `start_supervised` test tore down in 6 ms with no timeout.
- **`run/2` `[{^pid, sup, callbacks}] = :ets.take(...)` raising on a missing row** — the destructive-take/single-row contract is guaranteed by the sole caller `Runner.exec_on_exit` (one `register` → one `run` per pid); reachable only by internal misuse, which is out of scope.
- **`exec_on_exit_callbacks/3` returning `error || :ok`** — correct, not a demotion: the `run/2` `@spec` promises `:ok | {kind, reason, stack}`, and `exec_on_exit/3` (`runner.ex:541`) case-matches exactly those two shapes; returning bare `nil` would raise `CaseClauseError`, so the `|| :ok` is required.
- **`receive_runner_reply/4` timeout-kill branch leaving a late `{runner_pid, reason}` reply in the mailbox** — a genuine but well-known narrow race also present in shipped ExUnit; the stray message cannot match later fresh-ref receives, and the next callback spawns a new runner (`ensure_alive_callback_runner(nil, nil)`), so no result is corrupted.
- **`add/3` / `get_supervisor/1` crashing when the pid is absent** — caught by `rescue _ -> :error` and returned per the `:ok | :error` spec; no defect.
- **`Enum.reverse(callbacks)` ordering** — correct: `List.keystore` appends new refs, so reversing yields the documented LIFO on_exit execution order.

