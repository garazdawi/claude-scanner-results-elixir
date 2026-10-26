# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/gen_event.ex
<!-- sha256: 47c760d2f137d5d99b000acb3ec68d59aa54e7f0dc0cc75e4a61bbc581d8e413 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:11:08Z_

I have enough to write the report. The one concrete behavioral discrepancy is the `notify/2` asymmetry; everything else in the public API behaved correctly under test.

## Summary
`GenEvent` is a faithful, stable port of Erlang's `:gen_event`; the only behavioral wrinkle found is that `notify/2` raises on an unregistered atom-name manager while every other manager form silently returns `:ok`, but this mirrors the documented OTP replacement exactly.

## Bugfixes

#### `CSCAN-gen_event-notify-atom-name-raises`
- **Finding ID** — `CSCAN-gen_event-notify-atom-name-raises`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/gen_event.ex:243-249`, `GenEvent.notify/2` (the `is_pid/is_atom/tuple` clause)
- **Issue** — The `{:global, name}` (line 225) and `{:via, mod, name}` (line 234) clauses wrap delivery in `try/catch` and return `:ok` even when the target does not exist. The final clause — which handles `pid`, bare `atom` (a registered name), and `{name, node}` — calls `send(manager, {:notify, msg})` with no protection. `send/2` to an atom that is not currently registered raises `ArgumentError`/`:badarg`, so `notify/2` raises instead of returning `:ok` for that one manager shape, even though `notify` is documented as a fire-and-forget async call and `@spec`'d `:: :ok`.
- **Class** — Missing clause / error-handling asymmetry (unprotected delivery path).
- **Impact** — A caller that notifies a manager by registered name (e.g. a race where the notify arrives before the manager is started or after it has stopped) gets an unexpected exception rather than the best-effort `:ok` the other manager forms give. Callers treating `notify` as never-failing (its whole point) must add their own `try`. Dead pids and `{name, node}` do *not* raise, so the failure is easy to miss in testing and only surfaces for the atom-name form.
- **Evidence** — `verified`. `GenEvent.notify(dead_pid, :x)` → `:ok`; `GenEvent.notify({:some_name, :nonode@nohost}, :x)` → `:ok`; `GenEvent.notify({:global, :missing}, :x)` → `:ok`; `GenEvent.notify({:via, :global, :missing}, :x)` → `:ok`; but `try do GenEvent.notify(:unregistered_atom, :x) catch k,v -> {:raised,k,v} end` → `{:raised, :error, :badarg}` (crash trace: `:erlang.send(:nonexistent_name, {:notify, :x})` … `lib/gen_event.ex:247`). **Note the mitigating fact:** the documented replacement `:gen_event.notify/2` behaves identically — `:gen_event.notify(:unregistered_atom_xyz, :x)` also raised `{:raised, :error, :badarg}` in the same run — so this is inherited-from-OTP behavior, not an Elixir-introduced regression. It is reported for the internal asymmetry and the fire-and-forget contract mismatch, at Low severity accordingly.
- **Suggested fix** — For consistency with the other clauses, wrap the send in a catch: `try do send(manager, {:notify, msg}) catch _, _ -> :ok end; :ok` (or accept the OTP-consistent behavior and document that atom-name notify can raise). The smallest change that removes the asymmetry is the `try/catch`.

## Risks / caveats
- The module is entirely `@deprecated`/`@doc false`; every function is still exported and reachable, so I treated the exported functions and the `:gen_event` `@callback`s as the public surface. If "documented public API" is read to exclude `@doc false` functions, the single finding drops to informational.
- I exercised `start`, `add_handler`, `notify`, `sync_notify`, `ack_notify`, `call` (hit and `:not_found`), `which_handlers`, and `stop` against a real handler on the built-from-source Elixir 1.21.0-dev; all returned correct values. I did **not** dynamically exercise `swap_handler`/`swap_mon_handler`, `add_mon_handler` monitor/`:gen_event_EXIT` delivery, `system_*` sys-message callbacks, hibernation, or `GenEvent.Stream` process-handler collection — these were reviewed by reading only, so a defect confined to those paths would not have been caught.
- The record-index idioms (`handler(:id) + 1`, `handler(:ref) + 1`) and the `server_*` state threading were reviewed statically and matched across all call sites; I found no dropped-accumulator or wrong-index defect, but this rests on static reading, not exhaustive execution.

### Ruled out
- **`do_start/2` nil-vs-name branches call `:gen.start` with different arities** — Not a bug. `verified`: the `nil` branch uses `:gen.start/5` (no-name form) and the named branches use `:gen.start/6`; both arities exist in OTP `gen` and `GenEvent.start()`/`GenEvent.start(name: ...)` both succeed (smoke test started a manager and ran handlers).
- **`__using__` default `handle_call` always raises (the `1 -> {:remove_handler, ...}` clause is dead)** — Intentional, not a bug. `asserted` from the in-file comment "We do this to trick Dialyzer to not complain about non-local returns"; `:erlang.phash2(1,1)` is always `0`, so the raise is the real path and the second clause exists only to satisfy Dialyzer.
- **`call/4` `else {:ok, res} -> res` could miss other `:gen.call` shapes** — Ruled out. `verified`: `:gen.call/4` returns `{:ok, Reply}` on success and exits otherwise (caught by the `:exit` clause); `GenEvent.call/3` returned the handler reply and `{:error, :not_found}` correctly in testing.
- **`notify/2` multi-`when` guard (pid OR atom OR 2-tuple) leaves a manager form unmatched** — Ruled out. `asserted`+`verified`: the `manager` type is `pid | atom | {:global,_} | {:via,_,_} | {atom,node}`; `{:global,_}` and `{:via,_,_}` are matched by earlier clauses, and pid/atom/`{atom,node}` are covered by the guard, which the tests exercised for each shape.
- **`server_call`/`handle_down` off-by-one on `:lists.keyfind` position** — Ruled out. `verified`: `call` against a present handler returned its state and against an absent handler returned `{:error, :not_found}`, confirming the `handler(:id)+1` position is correct.

