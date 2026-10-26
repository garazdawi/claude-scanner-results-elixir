# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/registry.ex
<!-- sha256: 61311893cdea9ecc5f4b9806ba6ec9c0293608aad817faddaf69742d88124afa -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/registry.ex
<!-- sha256: 61311893cdea9ecc5f4b9806ba6ec9c0293608aad817faddaf69742d88124afa -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (first pass 2026-08-27T23:42:38Z; hedge-clearing pass 2026-08-28). SHA verified against the built source at `/opt/elixir/lib/elixir/lib/registry.ex` (`61311893…`, Elixir 1.21.0-dev / OTP 29)._

## Summary

`Registry.lock/3` acquires its partition lock with a `GenServer.call` that uses the default 5000 ms timeout instead of `:infinity`, so any caller waiting more than 5 s for a contended lock crashes with a `:timeout` exit. This finding is unchanged and remains backed by executed evidence (first-pass run; line 1854 re-confirmed this pass).

The first-pass reviewer flagged two hedges: **"did not exhaustively"** (the match-spec surface) and **a ruled-out entry whose Basis is `asserted` (3)** (the three `Ruled out` entries). This pass I stopped asserting and started *running*:

- **Entry 2 (`dispatch/4`)** and **entry 3 (`register_name/4`)** — I pushed the previously-unexamined inputs through a real build and compared outputs. Both are now `Not examined: none` and promoted `asserted → verified` on executed evidence.
- **Entry 1 (`values/3` vs `lookup/2`)** — I executed the differential axis (40 pids across all 4 partitions); it holds. The second axis ("partition count differs between write and read path") is **not constructible** through the public contract (partitions is one immutable value read from `key_info` on both paths), so it cannot be pushed; per the re-review rule it stays on `Not examined` and the entry **stays `asserted`**.
- **"did not exhaustively"** — I ran a `lookup`/`select`/`match`/`count_match` differential across three registry shapes; no discrepancy. The residual is narrowed but not closed (I did not fuzz `unregister_match` under concurrency or complex guards), so it stays a caveat, not a dismissal.

No findings dropped.

## Bugfixes

#### `CSCAN-registry-lock-call-timeout`

- **Finding ID** — `CSCAN-registry-lock-call-timeout`
- **Severity** — High
- **Confidence** — high
- **Location** — `registry.ex:1854`, `Registry.Partition.lock/3` (reached from public `Registry.lock/3` at `registry.ex:755-760`)
- **Issue** — Lock acquisition is `ref = GenServer.call(pid, {:lock, key})` with no timeout argument, so it defaults to 5000 ms. When the `lock_key` is already held, `handle_call({:lock, key}, ...)` (line 1924) enqueues the caller and returns `{:noreply, ...}`; the reply is only sent later by `go/2` (`GenServer.reply`) once the lock is released. If the current holder runs its function for longer than 5 seconds, the waiting caller's `GenServer.call` times out and raises an exit before it ever acquires the lock.
- **Class** — Timeout and lifetime (infinite-wait primitive given a finite timeout).
- **Impact** — `Registry.lock/3` is documented as a mutual-exclusion primitive: "Only one function can execute under the same `lock_key` at a given time." A lock exists precisely to serialize work that may be slow (the docs motivate it as an alternative to spawning a process when "copying the data to another process could be too expensive," lines 738-742). Any workload where the critical section can exceed 5 s — a large computation, IO, a slow downstream call — makes every *other* contender crash with `{:timeout, {GenServer, :call, ...}}` instead of waiting its turn. The lock silently stops being a lock under exactly the contention it is meant to handle, and the failure surfaces as an opaque GenServer exit in the caller, not an error the caller can act on.
- **Evidence** — `verified`. First-pass run (this same build): a holder task acquires `Registry.lock(LockReg, :k, fn -> sleep(7000) end)`, then the parent calls `Registry.lock(LockReg, :k, ...)` for the same key:
  ```
  holder has the lock; now the parent tries to acquire it (held > 5s)
  parent CRASHED after 5006ms: {:exit, {:timeout, {GenServer, :call, [#PID<0.100.0>, {:lock, :k}, 5000]}}}
  ```
  The parent never acquires the lock; it exits at the 5 s default deadline. Line 1854 re-confirmed unchanged this pass (`grep` at `registry.ex:1854`: `ref = GenServer.call(pid, {:lock, key})`).
- **Suggested fix** — Pass `:infinity` as the call timeout, matching every other unbounded-wait call in the module (e.g. `Task.await(&1, :infinity)` in `dispatch`): `ref = GenServer.call(pid, {:lock, key}, :infinity)`.

## Risks / caveats

- Behaviourally, I have now executed: the `lock/3` contention path (first pass), the duplicate-`:pid` `values`/`lookup` differential, the `dispatch/4` single- vs multi-partition serial/parallel differential, the `register_name/4` foreign-pid guard, and a `lookup`/`select`/`match`/`count_match` consistency differential (below). The residual unexamined surface is narrower than the first pass but non-empty: I did **not** execute `unregister_match/3` under concurrent registrations, nor exercise the match-spec guard-list path (`{:>, :"$1", 1}`-style guards) or `:"$_"`/`:"$$"` special variables. That is the honest boundary of what I ran; it is a stated gap, not a dismissal.

### Ruled out

Each entry lists which axes I actually pushed values through vs. which I did not. Per the re-review rule, reading a neighbour clause does not clear an axis; only executing an input through both sides does.

- **`values/3` and `lookup/2` partition selection for `{:duplicate, :pid}`** — `values/3` (lines 973-976) narrows to a single partition via `partition = hash(pid, partitions)` because it filters by a concrete `pid`, and `{:duplicate, :pid}` placement puts both key- and pid-partition at `hash(pid, partitions)` (`partitions/4`, lines 1661-1663) — so all entries for one pid live in exactly one partition. `lookup/2` (lines 695-698) has no pid to filter on and scans **all** partitions (`for partition <- 0..(partitions - 1)`). Both correct.
  - Basis: **`asserted`** (one axis executed, one not constructible).
  - Axes examined: **differential execution of registered-then-queried pids across a multi-partition duplicate-`:pid` registry** — *executed this pass.* Ran a `partitions: 4`, `keys: :duplicate` registry, 40 processes each registering key `"k"` with a distinct value; the pids' `phash2` landed in all four partitions `[0,1,2,3]`. Result: `Registry.lookup(R,"k")` (count 40) equalled the sorted union of `Registry.values(R,"k",pid)` over every pid (count 40); per-pid mismatches `= 0`. So `lookup` (all-partition scan) and `values` (single `hash(pid)` partition) agree entry-for-entry.
  - Not examined: **behaviour when the partition count differs between the writing and reading path** — *not constructible.* `partitions` is a single immutable value written into `key_info` at `start_link` and read by both the register and query paths; the public contract exposes no way to make the writer and reader see different counts. The axis cannot be pushed, so it remains and the entry stays `asserted`.

- **`dispatch/4` clause dispatch for single- vs multi-partition duplicate registries** — `key_info!` returns `{kind, 1, key_ets}` (non-nil) for one partition and `{kind, partitions, nil}` for many; every `keys` kind reaches a matching clause (lines 560-587), and the `{:duplicate, :pid}` branch selects serial vs parallel on `opts[:parallel]`.
  - Basis: **`verified`** — *Upgraded from `asserted` on executed evidence.*
  - Axes examined: **dispatch against an actual multi-partition duplicate registry** and **`partitions == 1` vs `> 1` compared on the same input** — *both executed this pass.* Populated key `"k"` with values `1..30` from 30 processes in three registries — `partitions: 1` (serial), `partitions: 4` (serial), `partitions: 4` (parallel) — and accumulated the values seen by the dispatch callback across all partition invocations. All three produced the identical sorted set, equal to `Enum.to_list(1..30)` (counts 30/30/30, all-equal `true`). The `partitions == 1` fast clause and the `> 1` serial/parallel recursion (`dispatch_serial`/`dispatch_parallel`, lines 592-625) deliver the same complete result.
  - Not examined: **none.**

- **`register_name/4` `when pid == self()` guard** — the single `defp register_name/4` clause (line 279) is guarded by `pid == self()`, reached via the public `register_name/2` `:via` callbacks (lines 276-277).
  - Basis: **`verified`** — *Upgraded from `asserted` on executed evidence.*
  - Axes examined: **an actual registration with a pid other than the one being registered** — *executed this pass.* Called `Registry.register_name({RN, "k"}, other_pid)` with `other_pid != self()`: it raised `FunctionClauseError` (caught). The contract-conforming call `Registry.register_name({RN, "k2"}, self())` returned `:yes`. So the guard behaves exactly as described. Note: a *non-conforming* caller gets a `FunctionClauseError` crash rather than `:no`; this is unreachable through the OTP `:gen`/`:via` convention (which invokes the callback inside the process being registered, so `pid == self()`), so it is not a bug for any contract-conforming path — but the observed behaviour on a foreign pid is a hard crash, recorded here rather than assumed.
  - Not examined: **none.**

## Did I verify this

- **`CSCAN-registry-lock-call-timeout`** — verified by execution (holder + contended caller, real build); line 1854 re-confirmed unchanged. Confidence: high.
- **Duplicate-`:pid` `values/3` vs `lookup/2`** — the primary differential axis is now **executed** (40 pids, all 4 partitions, `lookup == union(values)`, 0 mismatches). The remaining axis (differing write/read partition counts) is **not constructible** through the public API, so the entry stays `asserted`. Confidence in "no bug here": medium-high (up from medium — I now have an executed positive, just not a closed axis set).
- **`dispatch/4` single- vs multi-partition** — **executed** (p1 serial, p4 serial, p4 parallel all equal and complete). Promoted to `verified`. Confidence in "no bug": high.
- **`register_name/4` foreign-pid guard** — **executed** (foreign pid → `FunctionClauseError`; self → `:yes`). Promoted to `verified`, with the foreign-pid crash noted as by-design contract enforcement. Confidence in "no bug for conforming calls": high.
- **`{:duplicate, :key}` / `:pid` match-spec paths (`lookup`/`select`/`match`/`count_match`)** — **partially executed this pass** (previously "none / unexamined"). Ran a differential over three shapes (`:duplicate` p=1, `:duplicate` p=4, `{:duplicate, :key}` p=4): 24 entries under one key, values `{:tag, rem(i,3), i}`. In every shape `lookup == select == match(:tag,:_,:_)` (all count 24), and the filtered `match(:tag,0,:_)` and `count_match(:tag,0,:_)` both equalled the expected `rem==0` subset (8). No discrepancy. **Not** exercised: `unregister_match/3` under concurrency, guard-list conditions, and `:"$_"`/`:"$$"` special variables — so this stays an explicit caveat, not a dismissal. Confidence in "no bug": medium for the executed slice, none for the un-run slice.

### Dropped in re-review

- No findings dropped. The single `lock/3` finding is preserved unchanged.
- No dismissals were moved in the unsafe direction. Two ruled-out entries (`dispatch/4`, `register_name/4`) were upgraded `asserted → verified` **only** because I executed the previously-unexamined inputs, not on a re-read. The third (`values`/`lookup`) was left `asserted` because one of its axes is not constructible and therefore could not be pushed.
