# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/range.ex
<!-- sha256: c0b159472f6c7e833fae5c05ca4662a78cdb982f98150a3a661392eee08caa68 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:37:37Z_

## Summary

After brute-force differential testing of every arithmetic-bearing public function (`disjoint?/2`, `split/2`, `size/1`, `shift/2`, `to_list/1`) against naive reference implementations over large input domains, plus running all 40 doctests against the exact source under review, I found no genuine, reachable bugs in this file.

## Bugfixes

No findings

## Risks / caveats

- The installed `Range` beam and the source file under review were confirmed to agree: I recompiled `/Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/range.ex` directly with `Code.compile_file/1` and ran `doctest Range` against the freshly compiled module — all 40 doctests passed.
- Differential testing covered a bounded (but wide) integer/step domain, not the full integer space. The math in `disjoint?/2` (arithmetic-progression intersection via `Integer.extended_gcd`) and `split/2`/`split/4` is exact-integer and scale-invariant, so small-domain agreement is strong evidence, but a pathological large-magnitude interaction is not formally excluded — I judged this low-risk given the algorithms use no floats and no fixed-width arithmetic.
- The deprecated fallback clauses (`size/1` line 264, `to_list/1` line 432, `range?/1`) are only reachable by a map carrying `__struct__: Range` **without** a `:step` key — an old serialized representation, not the documented public API — so I did not treat their `nil`-step edge behavior as in scope.

### Ruled out

- **`disjoint?/2` intersection math wrong for some step/offset combination** — Ran `Range.disjoint?/2` against `MapSet.disjoint?` of the materialized element sets over all `f,l ∈ [-10,10]`, `step ∈ {±1..±5}` for both ranges (≈4.4M pairs): 0 mismatches. Basis: `verified`.
- **`split/2` violates its documented properties (first/last/step preservation, `size(input)==size(left)+size(right)`, or element concatenation) for some range/split** — Checked all four documented properties plus `to_list(left) ++ to_list(right) == to_list(input)` and the expected left-count semantics over `f,l ∈ [-5,5]`, `step ∈ {±1,±2}`, `split ∈ [-8,8]` (covering negative-beyond-size and past-end splits): 0 mismatches. Basis: `verified`.
- **`size/1` disagrees with actual element count (incl. empty ranges)** — `Range.size` matched both `length(Range.to_list/1)` and `length(Enum.to_list/1)` over `f,l ∈ [-12,12]`, `step ∈ {±1..±4}`: 0 mismatches; empty-range doctests (`1..10//-1`, `10..1//1`) pass. Basis: `verified`.
- **`shift/2` fails to translate every element by `steps_to_shift * step`** — `Enum.to_list(Range.shift(r, n))` equaled the manually shifted element list over `f,l ∈ [-6,6]`, `step ∈ {±1,±2,±3}`, `n ∈ [-6,6]`: 0 mismatches; the `shift(range, 0)` short-circuit returns the range unchanged. Basis: `verified`.
- **Misleadingly swapped parameter names in the second `split/4` clause (`defp split(last, first, ...)`) causing wrong output for decreasing ranges** — Names are swapped relative to the first clause but the body uses them self-consistently; decreasing-range splits (`5..1//-1`, `5..1//-2`, negative `split`) produce the documented tuples and passed the exhaustive `split/2` property sweep above. Cosmetic confusion only, not a defect. Basis: `verified`.
- **Deprecated stepless-map fallbacks (`size/1` L264, `to_list/1` L432) unreachable / crashing for real ranges** — The `first..last//step` struct pattern matches any `%Range{}` including one with an unusual step, so these fallbacks are reached only by maps lacking a `:step` key (legacy serialized form), which is their stated TODO purpose and outside the documented public API. Basis: `asserted` (pattern-semantics reasoning; not a claim of a defect but of non-scope).

