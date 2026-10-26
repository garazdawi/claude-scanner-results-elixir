# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/task.ex
<!-- sha256: e8cf64cde2854e3463ef6480931b09e86dac7ccb3b49e0b716e8b35ba09037a4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:54:32Z_

# Bug-hunting review — `lib/elixir/lib/task.ex`

## Summary
After enumerating candidate defects across every exported function and documented code path and verifying the suspicious ones against the built toolchain, I found no genuine, reachable bugs in this file.

## Bugfixes

No findings

### Ruled out

- **`async_stream/3` codepoint-count doctest (lines 723–726) sums to the wrong total** — Rejected. `"long string"` (11) + `"longer string"` (13) + `"there are many of these"` (23) = 47, and `Enum.sum_by/2` exists in this build. Basis: `verified` — ran the doctest verbatim, output `sum_by: 47`.
- **`await/2` and `await_many/2` doctests (lines 795–796, 973–978) mis-state results** — Rejected. Basis: `verified` — `await: 2` and `await_many: [2, 5]` observed.
- **`completed/1` uses a plain `make_ref/0` (not a monitor alias), so `demonitor(ref)` in `await`/`await_many`/`yield`/`ignore`/`shutdown` could crash or the reply be misrouted** — Rejected. `Process.demonitor/2` on a non-monitor reference is a legal no-op, and the reply is matched by the same ref, so completed tasks flow through every consumer. Basis: `verified` — `await_many([completed(:a), async, completed(:c)]) => [:a, :b, :c]`, `yield(completed(:z)) => {:ok, :z}`, `ignore(completed(:w)) => {:ok, :w}`, `shutdown(completed({:error,:x})) => {:ok, {:error, :x}}`.
- **`yield_many/2` `:limit` reached before timeout still triggers `:on_timeout` (would wrongly kill unreplied tasks)** — Rejected. The `yield_many(0, …)` base clause returns `{false, refs}`, and the `else` block guards unreplied entries with `_ when not timed_out? -> nil`, so `:kill_task`/`:ignore` never fire on a limit-satisfied return. Basis: `verified` — `yield_many(4 tasks, limit: 2, on_timeout: :kill_task) => [{:ok,1},{:ok,2},nil,nil]` (unreplied tasks left as `nil`, not killed).
- **`yield_many/2` leaks the `Process.send_after` timer / stray `timeout_ref` message on the `:noconnection` throw path (lines 1264–1266) and on the timeout path** — Rejected. The `:noconnection` and timeout-fired paths both end in `exit/1`, which terminates the caller, so an un-cancelled timer or unflushed message cannot be observed; the only path that returns to the caller (limit reached) cancels the timer and flushes `timeout_ref` in `yield_many(0, …)` (lines 1285–1288). Basis: `asserted` (control-flow reading — every non-return path calls `exit/1`); the return path's cancel+flush is present in-source.
- **`shutdown_receive/5` timeout→`:timeout_kill` recursion never matches the resulting `:killed` DOWN, hanging** — Rejected. After `Process.exit(task.pid, :kill)` the `:killed` DOWN falls through to the catch-all `{:DOWN, ^mon, _, proc, reason}` clause (line 1415), which demonitors and returns, so it terminates. Basis: `verified` — `yield_many(tasks sleeping ≥1s, timeout: 100, on_timeout: :kill_task)` returned `[nil, nil, nil]` promptly (each `shutdown(_, :brutal_kill)` completed).
- **Module doc (lines 257–260) mis-describes the `$callers` ordering (`"pid2 called pid_n", "pid2 was called by pid1"`)** — Rejected as a bug. For the canonical short list `[pid_n, pid2, pid1]` the prose is exactly correct (pid1 → pid2 → pid_n → current); the `...` only elides longer chains. Basis: `asserted` — narrative prose, internally consistent with the list layout it describes; no executable contract disagreement.
- **Clause ordering in `shutdown/2` (pid: nil vs. owner-mismatch, lines 1340–1346) could shut down a completed task from a non-owner** — Rejected. `shutdown(%Task{pid: nil}=task, _)` delegates to `ignore/1`, which itself performs the `owner != self()` check and raises, so cross-owner misuse still raises `ArgumentError`. Basis: `asserted` — `ignore/1` (lines 913–916) re-checks ownership before any receive.

## Risks / caveats
- The toolchain under `/opt/elixir` is `1.21.0-dev`; behavioral checks were run against it rather than a build pinned to this exact working-tree file, so any divergence between the checkout and that build is not covered — however, the `Task` public API exercised here is stable across these versions.
- Coverage was the file's documented public API (`start*/async*/await*/yield*/ignore/shutdown/completed/child_spec/__using__` and the deprecated `find/2`). Internal collaborators (`Task.Supervised.*`, `Supervisor.child_spec/2`) were treated as trusted and not reviewed; a defect delegated into those modules would not surface here.
- Concurrency-timing edge cases (e.g. a reply landing in the exact window between `Process.unlink` and `Process.exit` in `shutdown_send/2`) are guarded by the enforcer-process pattern and were reasoned about rather than stress-tested; no defect was found, but such races are inherently probabilistic to reproduce.

