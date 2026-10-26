# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/supervisor.ex
<!-- sha256: 67fe4de593b77ddacd48b17932afd63a2a43ab728972b0014c734728994a382d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:38:34Z_

## Summary

`Logger.Backends.Supervisor` is a small, internal (`@moduledoc false`) supervisor whose `add/1`, `remove/1`, and `start_link/1` flows match the upstream Elixir implementation and contain no triggerable defect.

## Bugfixes

No findings.

### Ruled out

- **Infinite recursion in `add/1` on `{:error, :already_present}` (lines 18–25).** I verified with a live Supervisor (script output above) that a *running* child yields `{:error, {:already_started, pid}}` (→ falls to `other`, no recursion), while only a *present-but-terminated* spec yields `{:error, :already_present}`. In that state `Supervisor.delete_child/2` succeeds and the recursive `add/1` starts fresh, so there is no loop. `verified` — see `terminate` → `:ok` then `start after terminate` → `{:error, :already_present}` in the run. (A pathological busy-loop is only theoretically possible if the child is momentarily `:restarting` and `delete_child` returns `{:error, :restarting}`; that state is self-resolving and matches upstream behavior — not an injected defect.)
- **`{:ok, child, info}` 3-tuple falling through the `case add(backend)` in `start_link/1` (lines 43–54), causing `CaseClauseError`.** The child is `Logger.Backends.Watcher`, a `GenServer`; `GenServer.start_link` never returns a 3-tuple, and a `Supervisor.start_child` 3-tuple only arises when the start function returns `{:ok, pid, info}`. So `add/1` cannot return `{:ok, _, _}` on this code path; unreachable. `asserted` (from the GenServer start contract) plus the live run showing only `{:ok, pid}` from a GenServer-backed child.
- **`{:error, error}` branch (lines 51–53) crashing `Exception.format_exit/1` on a non-exit-reason term.** A crashing transient child returns `{:error, {:boom, childspec}}` (verified: `crashing start` output). `Exception.format_exit/1` accepts any term and formats it; it does not raise. The branch therefore does what it intends — raise a descriptive install error — rather than crashing unexpectedly. `verified`.
- **`start_link/1` raising after `Supervisor.start_link` already succeeded, leaking the started supervisor.** The started supervisor is *linked* to the caller (the parent supervisor's `init`); a subsequent `raise` propagates and the linked supervisor exits with it. No orphaned process/leak; this is deliberate fail-fast-on-bad-backend behavior consistent with upstream. `asserted` (OTP link semantics).
- **`remove/1` (lines 28–37).** `terminate_child` then `delete_child`, returning `:ok` or the `{:error, _}` from `terminate_child`; return shapes are consistent and total for the documented Supervisor return values. No defect.
- **`init/1` restart intensity `max_restarts: 30, max_seconds: 3` (line 66).** A non-default but valid, convergent `:one_for_one` configuration; not a correctness bug.

## Risks / caveats

- The module is `@moduledoc false` and internal; its only documented callers are `Logger.Backends.Internal`. Findings are judged against those callers' expected return shapes, which I confirmed against a live `Supervisor`.
- I could not exercise the real `Logger.Backends.Watcher` start path end-to-end (that spans other files out of scope); the ruling that `add/1` never returns a 3-tuple rests on the GenServer start contract, which the Watcher uses. If a future Watcher change returned `{:ok, pid, info}`, the `{:ok, _}` clause in `start_link/1` would need widening.
- The locale warning (`native name encoding of latin1`) in the test run does not affect the logic exercised here.

