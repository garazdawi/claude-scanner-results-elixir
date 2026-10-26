# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/config.ex
<!-- sha256: f34133ca953616fe9ae04456af8246fe13e5c61f910adbe5ad5a4d57f9934e79 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/config.ex
<!-- sha256: f34133ca953616fe9ae04456af8246fe13e5c61f910adbe5ad5a4d57f9934e79 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T (siblings read: internal.ex, handler.ex, watcher.ex)_

All state-machine paths verified end-to-end (discard transition, periodic recovery, `configure/1`, unknown-message tolerance) with no crashes. In this pass I additionally read the three callgraph neighbours to close the cross-file hedges from the first pass. The cross-file invariants (shared `@counter_pos`, shared counter identity, sole-handler stderr sentinel, `fetch_env!` defaults) hold. No triggerable defect.

## Summary

`Logger.Backends.Config` is a small, faithful `:gen_event` handler whose documented entry point (`configure/1`) and callbacks all honor their contracts. Reading the siblings confirmed the overload-protection contract it participates in is coherent: the shared atomics counter is size-1 and both writers use position 1; `Config` reconciles the counter toward the manager's message-queue length while `Handler` increments it per event and is the sole party that actually discards. No triggerable bug was found.

## Bugfixes

No findings.

### Ruled out

- **`update_counter/2` counter drift / lossy `:counters.add(counter, @counter_pos, length - total)`** — Not a bug. The reconciliation is deliberately lossy (multi-paragraph comment, lines 62–73), and the threat model flags this as intentional. Confidence: high.\
  Axes examined: crash on flood; counter reconciles to `message_queue_len` after draining; interplay with the incrementing writer. | Not examined: none.\
  Executed evidence (carried from first pass, still valid): `:gen_event.add_handler` + flooding left the manager alive and the counter reconciled to `0` after draining. Cross-file confirmation this pass: the *other* writer, `Logger.Backends.Handler.threshold/1` (handler.ex:82–96), also uses `@counter_pos 1` (handler.ex:80) and only ever `:counters.add(counter, 1, 1)` — it increments, never reconciles — so `Config` is the sole reconciler and the "bumps may be lost" comment describes the true, intended interplay.

- **Shared `@counter_pos 1` atomics disagreement across files (cross-file invariant — first-pass hedge, now read)** — Ruled out. Confidence: high.\
  Axes examined: constant equality across writers; counter array size vs. position used. | Not examined: none (this is a static literal-equality fact, fully settled by reading, not an input-dependent behaviour).\
  `config.ex:60` declares `@counter_pos 1`; `handler.ex:80` declares `@counter_pos 1`; the counter is created in `internal.ex:139` as `:counters.new(1, [:atomics])` (size 1). Position 1 is the only valid index and both writers agree. **Upgraded from hedged assumption in first pass** ("cross-file invariants … out of scope for this single-file review") — now verified by reading the two co-writers.

- **Config receives a *different* counter than the Handler (cross-file invariant — first-pass hedge, now read)** — Ruled out. Confidence: high.\
  Axes examined: identity of the counter term threaded to each consumer. | Not examined: none.\
  `internal.ex:139` binds one `counter`, then passes the *same* term to `Logger.Backends.Watcher` → `Config`'s `init/1` (`internal.ex:147`) and to the handler config `config: %{counter: counter}` (`internal.ex:152–153`). Both `Config.init/1` (config.ex:18–19, via `load_state`) and `Handler.threshold/1` operate on the identical atomics reference. **Upgraded from hedged assumption in first pass.**

- **State machine never re-emits/recovers (stuck in `:discard`)** — Ruled out. Confidence: high.\
  Axes examined: discard→recover cycle with low threshold + fast periodic check; single-fire warn guard `if log == :log or periodic_check?`; no warn storm despite re-entrant `notify(self(), …)`. | Not examined: none.\
  Executed (first pass): exactly one "above :discard_threshold" on entry, one "below :discard_threshold" on recovery, then back to `:log`; manager stayed alive for a 10-message burst.

- **`warn/1` notifying `self()` causes re-entrancy / crash** — Ruled out. Confidence: high.\
  Axes examined: `self()` identity inside a gen_event callback; re-entrant `handle_event → update_counter(state, false)` hitting the fire guard. | Not examined: none.\
  Inside a `:gen_event` handler `self()` is the manager process, so `:gen_event.notify(self(), …)` (config.ex:103) broadcasts the warning to all handlers. Cross-file confirmation this pass: the re-broadcast event is also received by `Handler.log/2` (handler.ex:40) — which merely increments the shared counter and formats — so the re-entrant path is bounded and cannot storm; the `if log == :log or periodic_check?` guard is both-false once state is `:discard`. Executed (first pass): exactly one "above" warn for a 10-message burst.

- **`configure/1` / `handle_call` contract or return shape** — Ruled out. Confidence: high.\
  Axes examined: `:gen_event.call` reply value; env mutation; `{:ok, Reply, NewState}` shape; `Enum.each` over a `{key,value}` enumerable. | Not examined: none.\
  Cross-file note: `Internal.configure/1` (internal.ex:37–41) calls `Config.configure/1` with `Keyword.take(options, @backend_options)` — always a proper keyword list — so the `Enum.each(options, fn {key, value} -> …)` destructure (config.ex:35) is fed a conforming shape by its real caller.

- **Callback return-shape violations (`init`, `handle_event`, `handle_info`, `terminate`, `code_change`)** — Ruled out against the `:gen_event` behaviour. Confidence: high. Axes examined: each callback's declared return tuple. | Not examined: none. All valid; the handler added and ran without gen_event complaining.

- **`handle_info` crash on unknown message** — Ruled out. Confidence: high. Axes examined: catch-all clause totality; scheduled-message routing. | Not examined: none. `handle_info(_, state)` (config.ex:48–50) returns `{:ok, state}`; `handle_event(_event, state)` (config.ex:30) is likewise total.

- **`load_state/1` crash from missing app env (`fetch_env!`)** — Ruled out as a public-API bug. Confidence: medium.\
  Axes examined: which process/step sets the three keys; whether any in-folder caller can reach `load_state` before defaults exist. | Not examined: what a downstream operator app does if it deletes a stdlib default at runtime (out of this folder's control).\
  Cross-file confirmation this pass: neither `internal.ex` (which creates the counter and starts this handler) nor `config.ex` sets `discard_threshold` / `discard_threshold_periodic_check` / `utc_log`; `Handler.default_config/0` (handler.ex:27–36) *also* `fetch_env!`s the same keys, confirming these are Logger-application env defaults present before any handler is added. Reaching the crash requires deleting a stdlib default, not any public-API call.

- **`schedule_update_counter` timer leak / lifetime** — Ruled out. Confidence: high. Axes examined: re-arm site; count of simultaneously-pending timers; monitors/refs needing cleanup. | Not examined: none. `Process.send_after` is re-armed only inside `handle_info(@update_counter_message, …)` (config.ex:42–45), so exactly one timer is pending at a time; no refs/monitors created.

- **Sole-handler stderr sentinel silently suppressed by Config's presence (cross-file invariant — first-pass hedge, now read)** — Ruled out. Confidence: medium.\
  Axes examined: what handler-set `Watcher.logger_has_backends?/0` treats as "no backends"; whether `Config` adds a second always-present handler. | Not examined: the full set of handlers a host app may register at runtime (outside this folder).\
  The sentinel lives in `watcher.ex:70–76`: `:gen_event.which_handlers(Logger) != [Logger.Backends.Config]`. `Config` is exactly the single always-present handler the sentinel is written around; it does not register any additional persistent handler, so the stderr fallback logic in `watcher.ex` remains correct. **Upgraded from hedged assumption in first pass** ("the sole-handler stderr sentinel … lives there and is out of scope").

## Risks / caveats

- Verification exercised the module directly via `:gen_event` (the real manager registered as `Logger`) rather than through the deprecated `Logger.configure(discard_threshold: …)` app-level path; the module logic is driven exactly as the running system drives it.
- `@moduledoc false`: the module is internal, but `configure/1` and the `:gen_event` callbacks form its de-facto documented surface (invoked by `Logger.Backends.Internal`, confirmed at internal.ex:37–41).

## Did I verify this?

- Read all three callgraph neighbours named in the standing context: `internal.ex`, `handler.ex`, `watcher.ex`. This closed the first pass's three cross-file hedges — shared `@counter_pos`, shared counter identity, and the sole-handler stderr sentinel — each now grounded in a specific line in the neighbour (handler.ex:80, internal.ex:139/147/152, watcher.ex:72). These are static equality/identity facts settled by reading, not input-dependent behaviours, so no execution axis remains open for them.
- The behavioural ruled-out entries (counter drift, discard/recover cycle, `warn/1` re-entrancy) retain their first-pass executed evidence; the sandbox's Erlang toolchain reproduced them and I found no reason to overturn. I did not re-run them this pass because no code changed and no untried axis was identified.
- I did not execute a new end-to-end scenario that couples `Handler`'s per-event increment with `Config`'s reconciliation under concurrent load; that interplay is argued from reading both writers (medium confidence on the coupling), but each side individually is confirmed and the "lost bumps" comment documents the intended imprecision, so I did not upgrade it to a finding.

## Dropped in re-review

None. Every prior ruled-out entry is preserved; three cross-file hedges were upgraded from asserted assumptions to read-verified ruled-outs (marked "Upgraded from hedged assumption in first pass"). No prior finding was dropped, and no dismissal was moved from `asserted` to `verified` without either executed evidence (carried forward) or a static literal-equality fact confirmed by reading the co-defining line.
