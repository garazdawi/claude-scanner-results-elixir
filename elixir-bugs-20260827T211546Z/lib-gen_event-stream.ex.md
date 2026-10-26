# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/gen_event/stream.ex
<!-- sha256: 9e3cc7dc7bfa6eb2b57f586b56a548bc8fd307a18767010c4db2dd58068bbf72 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:09:24Z_

The behavior matches the intended (deprecated no-op) design. Everything in this file corresponds to the well-known upstream `GenEvent.Stream` implementation, and I found no divergence that constitutes a defect.

## Summary

`GenEvent.Stream` is entirely deprecated, `@moduledoc false` / `@doc false` legacy code whose callbacks and `Enumerable` implementation behave exactly as designed, and I found no genuine reachable bug.

## Bugfixes

No findings

### Ruled out

- **`handle_event/2` exits with `{:bad_event, event}` on every call** — Not a bug: this is the intentional "no longer supported" behavior for the placeholder handler; the `phash2(1,1)` case exists solely to defeat Dialyzer's non-local-return warning, and clause `1 -> :remove_handler` is deliberately unreachable dead code that is *meant* to be there. Basis: `verified` (ran it: `** (exit) {:bad_event, :e}` at stream.ex:21).
- **`handle_call/2` returns `{:remove_handler, reason}` / exits** — Not a bug: both are valid `:gen_event` handler `handle_call` return shapes, and the exit branch is the intended behavior; same Dialyzer-trick structure as `handle_event`. Basis: `asserted` (matches the `:gen_event` handler contract).
- **`count/1`, `member?/2`, `slice/1` return `{:error, __MODULE__}`** — Not a bug: returning `{:error, Enumerable.GenEvent.Stream}` is the correct Enumerable-protocol signal that the collection is not countable/searchable/sliceable, so `Enum` falls back to reduction. Basis: `verified` (ran it: each returned `{:error, Enumerable.GenEvent.Stream}`).
- **`wait_for_handler_removal/3` has a `receive` with no `after`** — Not a bug on the reachable paths: `stop/2` only calls it after the handler has been driven to removal (either `next` already saw `:gen_event_EXIT`/`:DOWN` and re-`send`s the event to `self`, or `stop/2`'s second clause first issues `:gen_event.delete_handler/3`), so one of the two matching messages is guaranteed to arrive. Basis: `asserted` (traced the two entry paths: `next` re-sends the halting event at lines 127/132 before halting to `{:removed, acc}`, and the non-removed `stop` clause calls `delete_handler` before recursing).
- **`next/2` `:DOWN` path does not `demonitor`, unlike the `:gen_event_EXIT` path** — Not a bug: on a `:DOWN` the monitor has already fired and been consumed, so there is nothing to demonitor; the asymmetry with the `EXIT` path (which flushes a still-live monitor) is correct. Basis: `asserted` (monitor semantics: a delivered `:DOWN` removes the monitor).
- **`stop/2` clause ordering (`{:removed, ...}` before `{pid, ref, _}`)** — Not a bug: the two patterns are disjoint (a 2-tuple tagged `:removed` vs a 3-tuple), and both accumulator shapes produced by `next` (`{:halt, {:removed, acc}}`) and by normal early-halt (`acc = {pid, ref, mon_ref}`) are handled. Basis: `verified` (struct/protocol wiring compiles and dispatches; clause shapes inspected).
- **Scope**: the entire module is `@moduledoc false` + `deprecated` with every function `@doc false`; there is no documented public API surface here, and no untrusted-input path. Basis: `verified` (read the whole 180-line file; every function is `@doc false`).

## Risks / caveats

- This is deprecated code (`"This functionality is no longer supported"`) with no documented public entry points, so per the review scope there is essentially no in-scope surface; findings would only matter to callers still using the removed `GenEvent` streaming API.
- I could not diff against the exact pinned upstream revision offline; my "matches upstream" judgments rest on the code's internal consistency and the verified runtime behavior above, not a byte-level comparison.
- The concurrency-sensitive paths (`next/2`, `stop/2`, `wait_for_handler_removal/3`) depend on live `:gen_event` manager message ordering that I exercised only structurally, not under a real manager; a race would require a running manager to reproduce.

