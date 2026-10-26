# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/map_set.ex
<!-- sha256: 1cda77077016cb598e0712fdc28353cc03856418cbdc7943f20496db51ada31e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:39:39Z_

All public paths, protocol implementations, and all 27 doctests behave correctly. This is a thin, well-covered wrapper over Erlang's `:sets` v2. Here is the report.

## Summary
`MapSet` is a thin, correct wrapper over Erlang `:sets` (version 2); every documented public function, protocol implementation, and all 27 doctests were exercised and behave exactly as documented, so no reachable bugs were found.

## Bugfixes

No findings

### Ruled out

- **`split_with/2` relying on `:sets` v2 internal shape** — `Map.split_with(map, fn {key, _} -> fun.(key) end)` treats the `:sets` v2 value as a plain map keyed by element. This is an internal-representation dependency but a valid one (v2 sets *are* maps), and the resulting sub-maps remain valid v2 sets. Verified: `MapSet.split_with(MapSet.new([1,2,3,4]), fn v -> rem(v,2)==0 end)` → `{MapSet.new([2, 4]), MapSet.new([1, 3])}`, and the empty-set case returns `{MapSet.new([]), MapSet.new([])}`. Basis: `verified`.
- **`symmetric_difference/2` disjoint fast-path** — the `if :sets.is_disjoint` branch (union) and the else branch (union of mutual subtracts) both compute the correct symmetric difference. Verified: overlapping `[1,2,3]/[2,3,4]` → `MapSet.new([1, 4])`; disjoint `[1,2]/[3,4]` → `MapSet.new([1, 2, 3, 4])`. Basis: `verified`.
- **`filter/2` and `reject/2` guard `is_function(fun)` (no arity check)** vs `split_with`'s `is_function(fun, 1)` — a mismatched-arity function would raise `BadArityError`, but only if the caller passes a function violating the documented `(a -> as_boolean(term))` spec; that is caller contract violation, out of scope. The truthiness coercion (`!!fun.(element)` / `!fun.(element)`) correctly handles `nil`/`false`. Verified via `filter`/`reject` doctests and `MapSet.filter(MapSet.new(["a", :b, "c"]), &is_atom/1)`. Basis: `verified`.
- **`new/1` idempotency clause** — `new(%MapSet{} = map_set)` returns the argument unchanged, and the enumerable clause dedupes via `:sets.from_list`. Both doctests pass. Basis: `verified`.
- **`equal?/2` using `===`** — set-of-`1` is not equal to set-of-`1.0`; verified `MapSet.equal?(MapSet.new([1]), MapSet.new([1.0]))` → `false`, matching the `@doc`. Basis: `verified`.
- **Enumerable/`slice`/`Collectable`/`Inspect` impls** — `Enum.slice/count/member?/reduce`, `in`, `Enum.into`, and `inspect/1` all produce documented output. Verified: `Enum.slice(MapSet.new([10,20,30,40,50]), 1..3)` → `[20, 30, 40]`; `inspect` → `"MapSet.new([10, 20, 30, 40, 50])"`. The `slice/1` single-arg `to_list` form is a valid `Enumerable.slice` return shape. Basis: `verified`.

## Risks / caveats
- Correctness of this module is delegated almost entirely to Erlang `:sets` v2; a semantic change or bug in the underlying OTP `:sets` implementation would surface here without any change to this file. Testing was done against the OTP/`:sets` version bundled with the elixir under review.
- `split_with/2` and the `Inspect` impl depend on `:sets` v2 being map-backed and on the private `Inspect.Algebra.to_doc_with_opts/2`; both are internal couplings that are correct today but would break silently if those internals changed. This is a maintenance risk, not a current reachable bug.

