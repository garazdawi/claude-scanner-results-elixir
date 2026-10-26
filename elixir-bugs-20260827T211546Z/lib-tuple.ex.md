# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/tuple.ex
<!-- sha256: 0d9ff60bedda35fb3c95c1eb85382efe8d4886acef1b71fc7a96c0f8cfb47711 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:56:23Z_

## Summary

`Tuple` is a thin, almost entirely correct wrapper over Erlang tuple BIFs; the only defect is a documented-error-type mismatch on the non-inlined path of `insert_at/3` and `delete_at/2` for negative indices.

## Bugfixes

#### `CSCAN-tuple-neg-index-wrong-exception`
- **Finding ID** — `CSCAN-tuple-neg-index-wrong-exception`
- **Severity** — Low
- **Confidence** — high
- **Location** — `tuple.ex:90` (`insert_at/3`) and `tuple.ex:117` (`delete_at/2`)
- **Issue** — The `@doc` for both functions promises: "Raises an `ArgumentError` if `index` is negative…". The clause guard is `is_integer(index) and index >= 0`, so a negative index has no matching clause. When the call is *inlined* by the compiler (the common direct-call case), the guard is bypassed and `:erlang.insert_element/delete_element` does raise `ArgumentError`, matching the doc. But when the function is reached without inlining — e.g. a function capture `&Tuple.insert_at/3` or `apply(Tuple, :delete_at, [t, -1])`, both fully public usages — the guard fails and the caller gets a `FunctionClauseError`, not the documented `ArgumentError`.
- **Class** — Documentation mismatch (contract violation: documented exception type)
- **Impact** — A caller who follows the documentation and writes `rescue ArgumentError` around a captured/applied `Tuple.insert_at`/`delete_at` will fail to catch the negative-index case and crash with an unhandled `FunctionClauseError`. Low cost because it requires both invalid (negative) input and the non-inlined call path.
- **Evidence** — `verified`.
- Inlined direct call: `elixir -e 'IO.inspect(try do Tuple.insert_at({:a,:b}, -1, :x) rescue e -> e.__struct__ end)'` → `ArgumentError` (guard bypassed by inlining).
- Non-inlined capture: `elixir -e 'f=&Tuple.insert_at/3; IO.inspect(try do f.({:a,:b}, -1, :x) rescue e -> e.__struct__ end); g=&Tuple.delete_at/2; IO.inspect(try do g.({:a,:b}, -1) rescue e -> e.__struct__ end)'` → `FunctionClauseError` / `FunctionClauseError`.

- **Suggested fix** — Make the guard-failure path itself raise the documented exception, e.g. add a fallback clause that raises `ArgumentError` for negative integers (or drop the `index >= 0` guard and let the BIF raise `ArgumentError` uniformly on both paths), so the exception type no longer depends on whether the call was inlined.

## Risks / caveats

- This is standard-library code that is heavily exercised; the "happy path" and the inlined error path are fully correct. The single finding is an inline-vs-non-inline exception-type inconsistency, deliberately called out because the `@doc` names the exception type as part of the contract. If you consider `FunctionClauseError` an acceptable substitute for `ArgumentError` on the (rare) captured/applied path, treat it as a documentation-precision note rather than a defect.
- I confirmed the doctests for `duplicate/2`, `insert_at/3`, `delete_at/2`, `sum/1` (including `{}` → `0` and `{255, 0.0}` → `255.0`), `product/1` (`{}` → `1`, `{255, 1.0}` → `255.0`), and `to_list/1` all pass by inspection and arithmetic; the `sum/product` empty-tuple base case (`sum(_,top,top,acc)` with `top = tuple_size + 1`) is correct.

### Ruled out
- **`insert_at`/`delete_at` index-greater-than-size** — doc promises `ArgumentError`; verified `Tuple.insert_at({:a,:b}, 5, :x)` raises `ArgumentError` on both paths (guard passes, BIF raises). Basis: `verified`.
- **`sum`/`product` on empty tuple** — `top` computed as `tuple_size(tuple) + 1 = 1`, base clause `sum(_,1,1,0)` matches immediately, returning the identity (`0`/`1`) as documented. No off-by-one. Basis: `verified` (doctests + arithmetic).
- **`sum`/`product` element-order / accumulator threading** — recursion increments `index` from 1 to `top-1`, threading `acc` correctly; verified `Tuple.sum({255,255})` → `510`, `Tuple.product({255,255})` → `65025`. Basis: `verified`.
- **`duplicate/2` negative/zero size** — guard `size >= 0` plus `:erlang.make_tuple`; `size 0` yields `{}`, negative is rejected; matches `@spec non_neg_integer`. No defect. Basis: `asserted` (guard is explicit and total for the spec'd domain).

