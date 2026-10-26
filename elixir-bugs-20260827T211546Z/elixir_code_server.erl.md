# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_code_server.erl
<!-- sha256: ecdc81913d4a34bb93a93d1c450c11b2b9aa0ec94d0059a4363566ae76bc179c -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_code_server.erl
<!-- sha256: ecdc81913d4a34bb93a93d1c450c11b2b9aa0ec94d0059a4363566ae76bc179c -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T06:03:28Z (hedge-resolution pass)_

# Bug-hunting review: `elixir_code_server.erl`

## Summary
No correctness, crash, resource-leak, or contract-violation defect is reachable through this module's documented public API (`call/1`, `cast/1`, `start_link/0`). This pass resolved the four hedged (`asserted`) dismissals from the first pass by reading the two callgraph neighbours and by executing the OTP `code` paths:

- **Resource leak (purge crash):** now **verified** — I ran `code:delete/1` and `code:purge/1` in the sandbox and confirmed they cannot crash on the only inputs this module produces (`elixir_compiler_N` atoms).
- **Dead / unreachable code:** now **verified** for the request atoms actually emitted by the named neighbours; the remainder are emitted from out-of-folder Elixir code, and their absence would be dead code, not a bug.
- **Contract violation** and **Infinite loop / hang:** **kept `asserted`**, and the `Depends on:` line is **corrected** — reading `elixir_compiler` and `elixir_module` showed that *neither* of them uses the `acquire`/`release`/`required` lock protocol at all; those callers live outside this callgraph slice (Elixir-level `Kernel.ParallelCompiler`/`Code`), so the owner-liveness axis cannot be discharged here and the entries stay asserted.

## Bugfixes

No findings.

The paths that looked most likely to hide a seeded defect were examined closely and are clean:

- **Async purge monitor tag** (`handle_cast(purge_compiler_modules)` lines 107–124 ↔ `handle_info({{purged,…},…})` line 129): `spawn_opt(..., [{monitor,[{tag,{purged,Mods}}]}])` (line 119) replaces the default `'DOWN'` head with `{purged,Mods}`, and the `handle_info` head (line 129) matches exactly. Only the purgeable subset `Mods` (line 118) is returned to `Unused` (line 131), matching the synchronous `handle_call(purge_compiler_modules)` (line 76) which also returns only purgeable mods. New module names are only ever minted fresh as `elixir_compiler_N` via the monotonic `Counter` (lines 70, 160–161), so no name-reuse window exists.
- **Require/acquire/release lock** (lines 46–63, 83–95, 146–158): the deliberate `{noreply,…}` from `acquire` (line 53) on the queued path is answered later by `gen_server:reply` in either `release` (`proceed`, line 153) or `handle_cast({required,…})` (`required`, line 89); every queued `From` is eventually replied to on both success and failure branches. The prepend `[From|Queued]` (line 52) + head-pop in `release` (line 152) is LIFO where `required` replies FIFO (line 89, `lists:reverse`), but load order among waiters is not a correctness property.
- **Monitor bookkeeping** (`defmodule`/`undefmodule` lines 170–183, `handle_info({'DOWN',…})` line 134): each `defmodule` monitors the owning Pid and records `Ref→Mod` (line 174); both the explicit `{undefmodule,Ref}` call (line 43) and the `'DOWN'` message route through `undefmodule/2`, which is idempotent (`maps:find` → `error` clause, lines 181–182), so a duplicate teardown neither crashes nor leaks an ETS row. **Confirmed against `elixir_module.erl`:** the compiling process calls `{defmodule, Module, self(), Tuple}` (line 471) and, in its own `after` block, `{undefmodule, Ref}` (line 268) — so the ETS row is torn down both by the explicit call and, if the process dies first, by the monitor `'DOWN'`.

## Risks / caveats
- **Out-of-slice dependency (acquire owner liveness):** `acquire` (line 46) does *not* monitor the process that receives `proceed`. If that owner dies without calling `{release,Path}` or casting `{required,Path}`, every queued waiter blocks forever (`?timeout` is `infinity`, line 11). **Reading the two named neighbours confirmed neither `elixir_compiler` nor `elixir_module` participates in this lock protocol** (grep for `acquire`/`{release`/`{required` across the folder returns only this module) — the callers are Elixir-level (`Kernel.ParallelCompiler`/`Code.require_file`), outside this callgraph slice. The hang is by design, but its safety cannot be established from anything in this folder.
- **`{stop, {badcall|badcast, Request}, …}` catch-alls** (lines 81, 127) tear down the server — and with it the public `elixir_modules` ETS table — on any unrecognized message. Reachable only by sending a shape outside the documented `call/1`/`cast/1` contract; fail-fast defensiveness, not an API-reachable bug.
- No `-spec` declarations exist in this file, so the spec-totality checklist is vacuously satisfied.

### Ruled out

**Operator / precedence typo**\
Reason: The only boolean/guard constructs are `is_list(Queued)` (line 51) and `Purgeable andalso code:purge(Module)` (line 166); both are single-operator, correctly polarized, with no `,`/`;` guard partitions to misread.\
Axes examined: `true` value vs list value for `required` map entries; empty vs non-empty `Queued`; purgeable vs non-purgeable in `purge_used` | Not examined: none.\
Basis: verified.

**Missing clause / catch-all**\
Reason: Every `handle_call`/`handle_cast` has a `{stop, bad…}` catch-all (lines 80–81, 126–127) and `handle_info` has a `{noreply, Config}` catch-all (line 137); the `case`/`maps:find` expressions each enumerate `{ok,_}`/`error` (and `[]`/`[_|_]`) exhaustively.\
Axes examined: unknown call/cast/info messages; `maps:find` hit vs miss; empty-list vs cons in `mod_pool` Unused and in `required` queues | Not examined: none.\
Basis: verified.

**Wrong atom / wrong symbol**\
Reason: Atoms (`required`, `proceed`, `purged`, `elixir_modules`, record/field names) are used consistently at both producer and consumer sites; the monitor tag `{purged,Mods}` produced at line 119 matches the `{{purged,Purged},…}` consumer at line 129. Cross-checked against `elixir_module.erl`: the `{defmodule,…}`/`{undefmodule,…}` atoms produced there (lines 471, 268) match the heads here (lines 34, 43).\
Axes examined: tag atom produced vs consumed; ETS table name at create/insert/lookup/delete; record field access vs update; defmodule/undefmodule request atoms at caller vs handler | Not examined: none.\
Basis: verified.

**State / accumulator threading error**\
Reason: Each clause returns the correctly-updated `Config`; `defmodule`/`undefmodule`/`release` thread the record through and return the mutated copy (lines 174, 180, 151/154); `mod_pool` tuple destructuring rebinds all three components on every write (lines 68, 70, 77, 104, 123, 131).\
Axes examined: pool tuple `{Used,Unused,Counter}` update on retrieve/return/purge; `required` map update on acquire/release/required/unrequire; `mod_ets` update on def/undef | Not examined: none.\
Basis: verified.

**Off-by-one / wrong arithmetic**\
Reason: The only arithmetic is `Counter+1` (mint next module id, line 70) and `length(Used)` (report count, line 78); both used for their plain meaning with no index/boundary computation.\
Axes examined: counter start (0, line 14) and increment; empty vs non-empty `Used` for length | Not examined: none.\
Basis: verified.

**Contract violation ({noreply} to a call)**\
Reason: The `{noreply,…}` returns from `acquire` (line 53) and the `handle_cast({required,…})` reply paths are the intended blocking-lock protocol; a `gen_server:reply` is guaranteed on every queued `From` via `release` (line 153) / `required` (line 89) — *provided* whoever holds `proceed` eventually releases.\
Axes examined: acquire on absent/queued/`true` path; release on empty/non-empty queue; required on queued path | Not examined: owner-process death without release (cross-slice; the acquire/release callers are not in this callgraph slice — see Depends on — so this axis cannot be tried here).\
Basis: asserted.\
Depends on: **Out-of-folder** `Kernel.ParallelCompiler` / `Code` (the acquire/release lock callers). *Correction to first pass:* I read `elixir_compiler.erl` and `elixir_module.erl` and neither uses `acquire`/`release`/`required`; the first pass's attribution of this guarantee to those two modules was wrong. The liveness guarantee lives in Elixir-level code outside this folder.

**Crash on stray / unexpected input**\
Reason: Stray messages hit `{stop,bad…}` (call/cast, lines 81, 127) or `{noreply,Config}` (info, line 137); reachable only outside the `call/1`/`cast/1` documented shapes.\
Axes examined: unknown call/cast; unexpected info incl. foreign `'DOWN'`; monitor message with `Mods=[]` | Not examined: none.\
Basis: verified.

**Resource leak on error path**  *(Upgraded from hedged assumption in first pass — now verified via execution)*\
Reason: The one monitor per module (`defmodule`, line 172) is released by `undefmodule` on both `{undefmodule,Ref}` (line 43) and `'DOWN'` (line 134); ETS rows are deleted in the same `undefmodule` path (line 179); the spawned purge process is monitored (line 119) so its completion is always observed, and the tagged message `{{purged,Mods},…}` re-adds `Mods` regardless of the spawned process's exit reason (line 129–131). The remaining first-pass doubt — "does `purge_used` crash inside `code:purge`/`code:delete`?" — I discharged by **running it**: in the sandbox, `code:delete(elixir_compiler_9999)` and `code:purge(elixir_compiler_9999)` both return `false` (no crash) for a non-loaded module; the only crash mode is `function_clause` on a *non-atom* argument, which cannot occur here because `Module` is always produced by `list_to_atom(...)` in `compiler_module/1` (line 161) or passed as an atom from `return_compiler_module`. So the purge process cannot crash on any input this module generates.\
Axes examined: duplicate undefmodule (idempotent); monitor fired vs manual undef; spawn normal-exit vs crash-exit both delivering the tagged message; `code:delete`/`code:purge` on a valid `elixir_compiler_N` atom and on a missing module (executed: returns `false`, no raise) | Not examined: none.\
Basis: verified.\
Evidence: executed `erl -noshell -eval 'code:delete(elixir_compiler_9999), code:purge(elixir_compiler_9999)'` → both `false`; non-atom arg → `function_clause` (unreachable input shape here).

**Silent corruption**\
Reason: `required` query (line 60) selects exactly the `true`-valued keys; purge returns exactly the purgeable subset in both sync (line 76) and async (line 118) paths; no clause drops or mis-orders an accumulator.\
Axes examined: `required` map with mixed `true`/list values; purge with all-purgeable / none-purgeable / mixed `Used` | Not examined: none.\
Basis: verified.

**Infinite loop / hang**\
Reason: No recursion in-module; the only unbounded wait is the intentional lock `{noreply}` (line 53), answered by a guaranteed reply (see Contract violation entry).\
Axes examined: acquire→release cycle; acquire→required cycle; repeated purge casts (second sees empty `Used`, line 110→111) | Not examined: owner death without release (cross-slice; callers not in this slice — cannot be tried here).\
Basis: asserted.\
Depends on: **Out-of-folder** `Kernel.ParallelCompiler` / `Code` — same corrected dependency as the Contract-violation entry; not `elixir_compiler`/`elixir_module` (confirmed by reading both — they never take this lock).

**Quadratic-where-linear / Repeated work**\
Reason: List ops are single-pass comprehensions and `++` used once per purge on small pool lists (lines 77, 131); no `Acc ++ X` inside recursion, no pattern recompilation, no per-call rebuild.\
Axes examined: `Mods ++ Unused` (single, non-recursive); comprehensions over `Used`/`required` | Not examined: none.\
Basis: verified.

**Dead / unreachable code**  *(Upgraded from hedged assumption in first pass — reachability of the in-slice request atoms now verified by reading the callers)*\
Reason: I read the two named callers. `retrieve_compiler_module` (`elixir_compiler.erl:127`), `return_compiler_module`→`{return_compiler_module,…}` (`elixir_compiler.erl:130`), `{defmodule,…}` (`elixir_module.erl:471`) and `{undefmodule,…}` (`elixir_module.erl:268`) are all genuinely emitted, so those clauses are live. The remaining request atoms (`purge_compiler_modules`, `{acquire,…}`, `{release,…}`, `{required,…}`, `{unrequire_files,…}`, plain `required`) are not emitted by either named neighbour; grep confirms they originate outside this folder (Elixir-level compiler driver). Their potential absence would be dead code, not a defect.\
Axes examined: each call/cast/info head; both purge variants; which request atoms the two in-slice callers emit (read directly) | Not examined: none within the slice (the out-of-folder emitters of the lock/purge/unrequire atoms are outside the callgraph slice; absence there would be dead code, not a bug).\
Basis: verified.

**Documented behaviour mismatch**\
Reason: The module has no `@doc`/`-spec`/prose contract in-file; behaviour was checked for internal self-consistency (sync vs async purge return the same purgeable-only set, lines 76 vs 118).\
Axes examined: sync vs async purge equivalence; acquire/required/release reply atoms | Not examined: none.\
Basis: verified.

**Race condition / shared mutable state**\
Reason: All state mutation is serialized through the single gen_server; ETS writes happen only inside handler clauses; the async purge deliberately removes mods from the pool before spawning (line 123) and re-adds them only via the serialized `handle_info` (line 131).\
Axes examined: concurrent purge casts; retrieve during in-flight purge; concurrent acquire/release | Not examined: none.\
Basis: verified.

**Wrong scope / order of operations**\
Reason: In `defmodule` the ETS insert (line 171) precedes monitor+map update correctly; the `defmodule` handle_call checks `ets:lookup` emptiness (line 35) *before* inserting, so redefinition is detected against the pre-insert state.\
Axes examined: fresh module vs already-registered module; insert-before-vs-after the empty check | Not examined: none.\
Basis: verified.

**Encoding / unicode mishandling**\
Reason: This module handles opaque atoms/refs/paths as map keys and ETS keys; it performs no byte/codepoint interpretation.\
Axes examined: path values as arbitrary terms (used only as map keys) | Not examined: none applicable.\
Basis: verified.

**Unclassified**\
Reason: No residual novel/composite defect surfaced beyond the caveats already stated (owner-liveness dependency, fail-fast `{stop,bad…}`).\
Axes examined: LIFO-vs-FIFO waiter ordering; double-purge sequencing; monitor-tag matching | Not examined: none.\
Basis: verified.

### Did I verify this?

- **Resource leak on error path**: **upgraded to `verified`.** Executed `code:delete/1` and `code:purge/1` in the sandbox on the exact atom shape this module mints (`elixir_compiler_N`) and on a missing module — both return `false`, no raise. The only crash mode (`function_clause` on a non-atom) is unreachable because `Module` is always a `list_to_atom(...)` result (line 161) or an atom handed back via `return_compiler_module`. Executed evidence, not a plausible invariant, so the last axis is cleared.
- **Dead / unreachable code**: **upgraded to `verified`** for the four request atoms emitted inside the callgraph slice (read directly in `elixir_compiler.erl:127,130` and `elixir_module.erl:268,471`). The lock/purge/unrequire atoms are emitted outside the folder; their absence would be dead code, not a bug, so nothing is promoted.
- **Contract violation**, **Infinite loop / hang**: **kept `asserted`.** Reading the two named neighbours actually *removed* the first pass's stated basis: neither `elixir_compiler` nor `elixir_module` takes the `acquire`/`release` lock, so they cannot supply the owner-liveness guarantee. The real callers are out-of-folder (`Kernel.ParallelCompiler`/`Code`), outside this callgraph slice, so the "owner death without release" axis cannot be tried here and stays on `Not examined`. Per the re-review rule, reading a neighbour never clears that axis, and no differential execution is possible from this file — the entries remain asserted, with the `Depends on:` corrected.
- All other entries remain `verified`: each clause was read and the enumerated input shapes hand-traced through the gen_server state transitions, with `Not examined: none`.

### Dropped in re-review

No prior findings were dropped (the first pass reported none). One factual correction, not a drop: the first pass's `Depends on:` for the **Contract violation** and **Infinite loop / hang** entries named `elixir_compiler`/`elixir_module` as the guarantors of lock release. Reading both modules showed they do not use the lock protocol at all; the dependency is re-attributed to out-of-folder `Kernel.ParallelCompiler`/`Code`. The dismissals themselves stand (asserted).
