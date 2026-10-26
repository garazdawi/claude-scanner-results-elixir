# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/supervisor/spec.ex
<!-- sha256: 9e989b1458a37285010cb2e1afc745523d7b681030261603b47c66de874dc263 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/supervisor/spec.ex
<!-- sha256: 9e989b1458a37285010cb2e1afc745523d7b681030261603b47c66de874dc263 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (hedge-resolution pass)_

The first-pass reviewer flagged the prior report's downstream-consumer hedge as "out of scope for this file." I resolved it by reading the actual consumer (`Supervisor.init/2` → `init_child/1` at supervisor.ex:785–854) and by re-executing every behavioural claim against a live Elixir/OTP toolchain. No hedge remains: every ruled-out entry below is now backed by either executed output I ran this pass or a neighbour-module read I performed this pass. No new findings resulted.

## Summary
`Supervisor.Spec` is a small, stable, deprecated child-spec builder whose three public functions (`supervise/2`, `worker/2,3`, `supervisor/2,3`) all behave exactly as their `@doc`/`@spec` describe. The 6-tuple child specs and the `{:ok, {flags, children}}` supervise result are consumed unchanged by the supervisor front-end, which I confirmed by reading `Supervisor.init/2`/`init_child/1`. I found no reachable bug.

## Bugfixes

No findings

### Ruled out

- **`supervise/2` returns the wrong shape vs `@spec :: {:ok, tuple}`** — Returns `{:ok, {{strategy, maxR, maxS}, children}}`, a valid `{:ok, tuple}`. Executed this pass: `{:ok, {{:one_for_one, 3, 5}, [...]}}`. This is exactly the classic OTP `{ok, {SupFlags, ChildSpecs}}` init shape (3-tuple `{strategy, intensity, period}` flags) that `:supervisor` accepts when returned from a `c:Supervisor.init/1` callback. Basis: `verified`. Axes examined: output shape, flags arity, `:ok` wrapper | Not examined: none.
- **`supervisor/2` default shutdown not `:infinity`** — `Keyword.put_new(options, :shutdown, :infinity)` runs before `child/4`'s `5000` default, so `:infinity` wins. Executed this pass: `{MySup, {MySup, :start_link, [1]}, :permanent, :infinity, :supervisor, [MySup]}`. Basis: `verified`. Axes examined: put_new precedence vs child default, override path | Not examined: none.
- **`worker/3` default shutdown/options** — Doc claims `shutdown: 5000`, `function: :start_link`, `restart: :permanent`, `modules: [module]`, `id: module`. Executed this pass with a `restart: :temporary` override: `{MyWorker, {MyWorker, :start_link, [1, 2]}, :temporary, 5000, :worker, [MyWorker]}` — override respected, all four other defaults correct. Basis: `verified`. Axes examined: each of the five defaults, one override | Not examined: none.
- **`assert_unique_ids/1` duplicate detection** — The O(n²) `id in rest` scan raises on a repeated id, returns `:ok` otherwise. Executed this pass (two `worker(A, [])`): `"duplicate ID A found in the supervisor specification, please explicitly pass the :id option ..."`. Basis: `verified`. Axes examined: dup present, dup absent | Not examined: none.
- **`get_id/1` on a non-6-tuple (e.g. a v1.5 map spec)** — Falls through to the catch-all clause (spec.ex:190) raising a descriptive `ArgumentError` naming `Supervisor.init/2`. Executed this pass by pushing a real map spec `%{id: :x, start: {A, :f, []}}` through `supervise/2`: `"invalid tuple specification given to supervise/2. ... use Supervisor.init/2 instead of Supervisor.Spec.supervise/2. ... Got: %{id: :x, start: {A, :f, []}}"`. This is the documented contract for misuse, not a crash-on-valid-input. Basis: `verified` (upgraded from first-pass `asserted` — I executed the fallthrough path this pass rather than reasoning about the clause literal). Axes examined: map input, 6-tuple input | Not examined: none.
- **`supervise/2` missing/`nil` `:strategy`** — `if !(strategy = options[:strategy])` raises `"expected :strategy option to be given"`, matching the "required" doc. Executed this pass: `nostrat: "expected :strategy option to be given"`. Basis: `verified`. Axes examined: `nil` strategy, present strategy | Not examined: none.
- **`modules(GenEvent)` special-case** — Returns `:dynamic` for `GenEvent`, `[module]` otherwise. Executed this pass: `{GenEvent, {GenEvent, :start_link, []}, :permanent, 5000, :worker, :dynamic}` for the GenEvent path, and `[MyWorker]` for the ordinary path. Basis: `verified`. Axes examined: GenEvent clause, non-GenEvent clause | Not examined: none.
- **6-tuple / `{:ok, {flags, children}}` output rejected or mis-consumed by the supervisor** *(resolves the first-pass "downstream OTP behavior is out of scope" hedge)* — Verified this pass by reading the neighbour consumer. `Supervisor.init/2` (supervisor.ex:785) maps each child through `init_child/1`; the clause `defp init_child({_, _, _, _, _, _} = tuple), do: tuple` (supervisor.ex:839–841) accepts the exact 6-tuple `child/4` emits and forwards it verbatim, and the error clause (supervisor.ex:843–854) explicitly lists "a tuple with 6 elements generated by Supervisor.Spec (deprecated)" as supported. The `supervise/2` result path returns `{:ok, {{strategy, maxR, maxS}, children}}` directly to OTP `:supervisor` as its init value (the 3-tuple `sup_flags` classic form), bypassing `init_child/1` entirely; its children are `child/4`'s 6-tuples, which OTP `:supervisor` accepts as old-style child specs. The consumer imposes no field-shape constraint the builder can violate. Basis: `verified` (neighbour read performed this pass; the shapes it accepts were also executed above). Note: reading a neighbour tells me what its clauses accept, not which unusual inputs might disagree — but here the builder's *only* outputs are the two shapes I executed, and both match a literal consumer clause, so there is no untried input axis on the builder side.

## Risks / caveats

The file has no doctests (the `## Example` blocks lack `iex>` prompts), so there is no executable-doc surface to diverge from the code. Review covered all three public entry points and their private helpers (`child/4`, `get_id/1`, `assert_unique_ids/1`, `modules/1`). The sole downstream question the first pass left open — whether the produced tuples are actually accepted by the code that consumes them — is closed by reading `Supervisor.init/2`/`init_child/1` (supervisor.ex:785–854).

## Did I verify this
- Read the full target file (spec.ex:1–293).
- Read the downstream consumer `Supervisor.init/2` and `init_child/1` (supervisor.ex:780–854) this pass to confirm the 6-tuple and `{:ok, {flags, children}}` shapes are consumed correctly — replaces the first-pass "out of scope" hedge with a read-verified conclusion.
- Executed all seven behavioural claims this pass on a live Elixir/OTP toolchain (`elixir t.exs`, `+fnu`): default worker/supervisor shutdowns, restart override, GenEvent `:dynamic`, duplicate-id raise, missing-`:strategy` raise, non-6-tuple (map) fallthrough raise, and the `supervise/2`/GenEvent result tuples. Outputs are quoted in the ruled-out entries above.

### Dropped in re-review
None. Every prior ruled-out entry is preserved. One entry (`get_id/1` non-6-tuple) was moved from `asserted` to `verified` because I executed the fallthrough path this pass with a real v1.5 map spec — a tightening backed by executed evidence, not by re-reading the clause. The first pass had no findings to drop, and its lone hedge was resolved by verification (option a), not upgraded to a finding, because reading the consumer confirmed the contract holds rather than revealing a gap.
