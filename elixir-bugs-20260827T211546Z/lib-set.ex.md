# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/set.ex
<!-- sha256: 4bac170c9919334f2ca2b4db77a4c072731e7f5b2e70887f40ad44b666846396 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:43:41Z_

All same-type operations return correct results and the code matches upstream Elixir faithfully. The cross-type path can't be exercised on this build because `HashSet` has been removed (only `MapSet` remains), but the reduce-based fallback logic is standard and correct by inspection. I found no defect.

## Summary
`Set` is a faithful, correct deprecated dispatch wrapper over set-implementation modules (now effectively only `MapSet`); every public function behaves as documented and matches upstream Elixir.

## Bugfixes

No findings

## Risks / caveats

- **Search space:** I reviewed all 12 exported functions (`delete`, `difference`, `disjoint?`, `empty`, `equal?`, `intersection`, `member?`, `put`, `size`, `subset?`, `to_list`, `union`), the two private helpers (`do_subset?`, `unsupported_set`), and the `target/1` macro. I verified the same-type dispatch path and the `Enumerable.reduce/3` accumulator threading (`|> elem(1)` correctly extracts the accumulator from `{:done, acc}`/`{:halted, acc}`). All produced correct results — `verified` (ran the public API against `MapSet`: diff `[1]`, union `[1,2,3,4]`, inter `[2,3]`, disjoint `true`, subset `true`, equal `true`).
- **Cross-target path not runtime-exercised:** The `else` branches (differing `target1`/`target2`) require two distinct struct-based set implementations both exporting these callbacks and `Enumerable`. On this build (`1.21.0-dev`) `HashSet` is removed, so only one implementation (`MapSet`) exists and the cross-type branches are effectively dead in practice. I judged them correct by inspection against upstream — `asserted` for that specific path — but note the semantics (`difference` deletes set2's members from set1; `equal?` uses size-then-subset; `do_subset?` halts `false` on first non-member) are the standard, correct definitions.
- The module is deprecated (`@moduledoc deprecated:` + per-function `@deprecated`); it exists only for backward compatibility and callers are steered to `MapSet`. There are no doctests to contradict.

