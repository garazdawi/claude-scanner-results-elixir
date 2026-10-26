# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/server.ex
<!-- sha256: fe83b142c94c4da21bfc6295fc5ff2f7f7ec0fd411ba23c6a50f35fda86b4186 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:37:32Z_

## Summary
`ExUnit.Server` faithfully implements the module-scheduling state machine and, across the tricky count/group/queue paths, behaves correctly; I found no defects reachable through its public API.

## Bugfixes

No findings

## Risks / caveats
- The module is `@moduledoc false`; its "public API" (`add_module/2`, `modules_loaded/1`, `take_async_modules/1`, `take_sync_modules/0`, `restore_modules/2`) is really an internal contract driven only by `ExUnit.Runner`/`ExUnit.Case`. Several `=` bindings (`take_sync_modules`'s `%{waiting: nil, loaded: :done, async_groups: []} = state`, `OnExitHandler`'s destructive `:ets.take`) are deliberate assertions that raise on out-of-order calls; these are contract preconditions, not bugs, and are out of scope per the misuse-exclusion rule.
- I drove the real compiled module through its public API and did not exercise the `--repeat-until-failure` path (`restore_modules/2`) with a live subsequent take; I inspected it by reading and it round-trips the taken async/sync/group structures correctly, but that specific claim is asserted, not behaviorally verified.

### Ruled out
- **`take_until/3` off-by-one when `count > queue size`** — the deliberately hand-rolled replacement for `:queue.split` (which raises when count exceeds size). Verified: `take_async_modules(3)` on a 5-item queue returned exactly `[a,b,c]`, and `take_async_modules(10)` on the 2 remaining returned `[d,e]` then `nil` — no crash, no drop, correct min(count,size) semantics.
- **Variable shadowing in the `async_groups` branch of `take_modules/1`** (`groups` rebound from the split-name-list to the reduced leftover map). Verified: grouped module `:g1` was returned as `[g1: [g1m1, g1m2]]` in insertion order, and the subsequent take returned `nil`, proving `state.groups`/`async_groups` were updated to the correct remainders and `Map.pop!` did not double-pop.
- **Group entry ordering (stored reversed via `Enum.reverse(names, entries)`, un-reversed on take)** — verified the two group members came back in original insertion order (`g1m1, g1m2`).
- **`modules_loaded` reversing/dedup logic** — read against known upstream; the reverse of `async_groups`/`sync_modules` and the queue-preserving `Enum.uniq` for the `uniq?` path match the take-side ordering, and the sync path returned `[s1]` correctly after all async were drained. No divergence found.
- **Missing catch-all `handle_call`** — the `{:add, _, _, _}` fallthrough clause (line 171) returns `:already_running` for any add after `loaded: :done`, so post-start adds surface as the documented raise in `add_module/2` rather than a `FunctionClauseError`; the other calls are only issued by the runner in-order.

