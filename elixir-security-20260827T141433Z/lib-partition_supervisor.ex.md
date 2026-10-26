# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/partition_supervisor.ex
<!-- sha256: d6dff430ebf8eb94af6e8f91870555cbe44379be410bac03b1e31f46c994841e -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/partition_supervisor.ex
<!-- sha256: d6dff430ebf8eb94af6e8f91870555cbe44379be410bac03b1e31f46c994841e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

# Security review — `partition_supervisor.ex`

## Summary
`PartitionSupervisor` is a supervision/routing utility whose only untrusted-data path — the routing `key` in the documented `{:via, PartitionSupervisor, {name, key}}` API — is handled with total, bounded operations (`rem/2`, `:erlang.phash2/2`), so no security defect reachable through its documented public API was found. This re-review discharges the two hedges the first pass carried (routing-primitive trust; the `Process.get(:ets_table)` / OTP-restart coupling) by executing both against the in-repo build rather than reasoning about them. No findings were upgraded; no findings were dropped.

## Security

`No findings`

The file was inventoried against every Phase-1 sink class. It contains no code evaluation, no atom creation from external bytes, no path/file/archive/command/network handling, no deserialization, no crypto, and no Unicode/template work. The only value that crosses a trust boundary through a documented entry point is the routing `key`; it was traced to its sinks and exercised with hostile inputs (below). Everything else is developer-supplied configuration (child spec, `:partitions`, `:with_arguments`, name), which the scope rules treat as trusted.

### Ruled out

- **Routing `key` → out-of-range partition / crash (via-tuple API).** In `whereis_name/1` (line 534) the key becomes `rem(abs(key), partitions)` for integers and `:erlang.phash2(key, partitions)` otherwise. Both are total functions on any term yielding `0..partitions-1`; BEAM integers are bignums so `abs/rem` cannot overflow.\
  `Basis: verified.` Confidence: high. Executed a property check over 7 partition counts `{1,2,3,4,7,64,1000}` × 19 key shapes — `0`, `1`, negatives, a `±(1<<<4000)` bignum of each sign, `3.14`, `-0.0`, an atom, a 100 KB binary, a tuple, a nested list, a pid, a ref, a map, and a fun — computing the exact routing expression; every result was in `0..p-1` (0 out-of-range). First pass additionally exercised these against a live 4-partition supervisor with no crash.\
  Axes examined: integer sign (incl. huge bignum both signs), non-integer term type (atom/binary/tuple/list/pid/ref/map/fun/float incl. `-0.0`), partition-count magnitude. Not examined: none.
- **`Process.get(:ets_table)` coupling across OTP restart — routing gap after a child restart (upgraded scrutiny of first-pass hedge).** `start_child/4` (line 314) calls `register_child/2` (line 329), which writes `:ets.insert(Process.get(:ets_table), …)`. This requires the supervisor process's dict entry set by `init/1` (line 337) to be present when the OTP supervisor drives a *restart*, not just the initial start.\
  `Basis: verified.` Confidence: high. Executed: started an atom-named `PartitionSupervisor` (4 partitions), `Process.exit(pid, :kill)` on the partition-0 child, awaited `:DOWN`, then re-read ETS row `0`: it advanced from `{0, #PID<0.103.0>}` to `{0, #PID<0.107.0>}` and `GenServer.call` via `{:via, …, {MyPS, 0}}` reached the new pid. This proves the restart-driven `start_child/4` ran in the same process that holds `:ets_table` and re-registered the child — the coupling holds on the OTP restart path, not only the initial-start path.\
  Axes examined: initial-start path vs. restart-after-crash path (process-dict availability in the supervisor process during restart). Not examined: none for the supervisor-driven paths — `start_child/4` is `@doc false` and reachable only via the generated child spec `{__MODULE__, :start_child, …}` (line 296), which only the OTP supervisor process invokes; no public entry point calls it from another process.
- **`:public` named ETS routing table (`init_table/1`, line 342) — routing hijack / read of pids by another process.** `Basis: verified.` Confidence: high. First pass confirmed `:ets.info` reports `protection: :public, named_table: true`; an arbitrary co-resident process can `:ets.insert(name, {0, other_pid})` to redirect partition 0. **Out of scope**: not reachable through this file's documented public API (requires direct `:ets` calls from co-resident code), and a BEAM node is a single trust domain — code that can call `:ets.insert` can already `:erlang.halt/0` and read every table. The `:public` level is deliberate: `resize!/2` (lines 398, 402) writes the `:partitions` entry from the *caller's* process, not the supervisor's. Standard practice (mirrors `Registry`), not a defect.
- **No upper bound on `:partitions` (line 272) — mass process creation.** `partitions` is only floor-checked (`>= 1`), so a large value starts that many children via `for partition <- 0..(partitions-1)` (line 289). Developer-supplied config, not data forwarded from an untrusted boundary; does not cross a trust boundary under the scope rules. Confidence: high.
- **`phash2`/`rem` cost on a giant routing key — DoS.** Cost scales with the size of the term the caller already constructed; no amplification. `Basis: verified.` Confidence: high — the 100 KB binary and nested-list keys in the routing property check returned instantly.
- **`resize!/2` concurrency (lines 378–419).** Mutation wrapped in `Registry.lock(@registry, supervisor, …)`, serializing concurrent resizes; the range check `partitions not in 0..max//1` (line 380) bounds the loop to the original partition count, and the `partitions == 0` guard in `whereis_name/1` (line 530) turns a resized-to-zero supervisor into an `ArgumentError` rather than corrupt state. `Basis: verified` (range/zero behavior exercised first pass). Confidence: medium — the serialization guarantee itself rests on `Registry.lock/3` semantics (stdlib, treated as trusted per scope), which I did not re-derive; the in-file range/floor checks that bound the loop are verified by reading.
- **`ensure_registry/0` start race (line 352).** Two supervisors can both `Supervisor.start_child(:elixir_sup, {Registry, …})`; the loser gets `{:error, {:already_started, _}}`, which is ignored, and the registry runs either way. Benign. Confidence: high (control flow is local to this file).
- **Error-message `inspect/1` (lines 274, 282, 304, 382, 394, 410, 531).** All interpolate developer-supplied config (partition count, `with_arguments` fun, supervisor name) or supervisor error reasons into `ArgumentError`/`RuntimeError` messages — no value forwarded from a network/disk/dependency boundary, so no secret-leak or log-injection surface. Confidence: high.
- **Atom handling (`child_spec/1`, `init_table/1`).** `name`/`id` are passed through as already-existing terms; `:ets.new(name, [:named_table])` uses an existing atom and does not create one. No `String.to_atom`, `Module.concat`, or equivalent. Confidence: high.

## Risks / caveats

- **Non-security latent behavior (not a finding).** Confidence: high. If a partition's child returns `:ignore` or an error, `start_child/4` (lines 314–327) skips `register_child`, leaving no ETS entry for that partition while the `:partitions` count still includes it. Subsequent routing to that partition hits `:ets.lookup_element/3` (line 537) and raises in the *caller*. Driven by the developer's child spec, not by untrusted data — an availability edge, not a security issue.
- **Residual trusted dependencies (scope, not a hedge).** Confidence: high that these are out of scope. Routing/locking correctness ultimately rests on `Registry.lock/3`, `Supervisor.*`, and `:erlang.phash2/2`. These are Elixir stdlib / ERTS internals that the folder scope rules treat as trusted; they are not audited here by design. This is a scope boundary, not an unverified assumption about *this* file's behavior — the in-file consequences that depend on them (routing range, restart re-registration, resize loop bounds) were each executed against the build above.
- Empirical tests used the in-repo build (Elixir 1.21.0-dev, OTP 29, ERTS 17.0.2); results reflect that build's actual behavior.

## Did I verify this?

- Routing stays in `0..partitions-1` for all key types and partition magnitudes — **yes, executed** (property check, 0 out-of-range across 133 combinations).
- Restart-driven `start_child/4` re-registers the new child (the `Process.get(:ets_table)` / OTP-restart coupling the first pass left as "did not exhaustively enumerate") — **yes, executed** (killed a child, observed the ETS row advance to the new pid and routing succeed). Hedge discharged.
- `phash2`/`rem` DoS amplification — **yes, executed** (100 KB and nested keys returned instantly).
- `:public` ETS reachability and `resize!` range/zero guards — **yes** (ETS protection inspected; range/zero behavior exercised first pass; verified again by reading lines 380/530).
- `Registry.lock/3`, `Supervisor.*`, `phash2/2` internal implementations — **no, by scope** (trusted stdlib/ERTS; their in-file consequences are executed, but their internals are not audited).

## Dropped in re-review

None. Every ruled-out entry and caveat from the first pass is preserved; the two first-pass hedges ("did not audit [routing primitives]" and "did not exhaustively enumerate every OTP restart scenario") were converted to `Basis: verified` with cited executed evidence, not dropped. No entry was upgraded to a finding, because the axis that the restart hedge left untried (restart-path process-dict availability) was tried and passed.
