# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/map.ex
<!-- sha256: cfae2326e8c87ad0fe10400dec3eae7e3c7bd90ebbde85d7d51a3c71af02387f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:39:24Z_

All documented behaviors and doctests I exercised match the code exactly. This is the canonical `Map` standard-library module; my review confirms it is clean.

## Summary
No genuine, reachable bugs were found in `map.ex`; the module is the canonical Elixir `Map` implementation and its public API behaves as documented.

## Bugfixes

No findings

### Ruled out

- **`replace!/3` / `update!/3` KeyError message** — `:maps.update/3` raises an Erlang `{badkey, Key}`, but Elixir normalizes it to a `KeyError` with the documented message. Verified: `Map.replace!(%{"foo" => "bar"}, "unknown", ...)` produced `** (KeyError) key "unknown" not found in: %{"foo" => "bar"}`, matching the doctest. Basis: `verified`.
- **`pop/3` / `pop!/3` / `pop_lazy/3` default handling** — all three correctly distinguish present (`{_, _}` from `:maps.take`) from absent (`:error`). Verified `Map.pop(%{"a" => 1}, "b", 3)` → `{3, %{"a" => 1}}`. No shape confusion between `:maps.take/2` returning `{Value, Map2}` vs `:error`. Basis: `verified`.
- **`new/1` clause ordering** — `%_{}` (struct) precedes `%{}` (plain map), so structs route to `new_from_enum` and plain maps pass through unchanged; there is no unreachable struct clause. Basis: `asserted` (clause order is visually decisive) plus `Map.new([{:b,1},{:a,2}])` and `Map.new(a: 1, a: 2, a: 3)` verified to match doctests.
- **`get_and_update/3` `:pop` path** — returns `{current, delete(map, key)}` where `current` was read before deletion; the `{get, update}`, `:pop`, and catch-all `other` branches cover the full documented contract, and the catch-all raises with an informative message. Basis: `verified` (`Map.get_and_update(%{a: 1}, :b, fn v -> {v, "x"} end)` → `{nil, %{b: "x", a: 1}}`).
- **`split_with/2` truthiness** — uses `if fun.(...)`, correctly treating only `nil`/`false` as the "false" bucket; matches the documented "truthy/falsy" split. Verified against the first doctest. Basis: `verified`.
- **`filter/2` / `reject/2` complementarity** — `do_filter` keeps on truthy, `do_reject` keeps on falsy; the two are exact complements with no shared/dropped element. Basis: `asserted` (reading the two recursions), consistent with the `intersect`/`split_with` runs performed.
- **`take/2`, `drop/2`, `split/2` non-list/non-map paths** — each has the documented deprecation warning for non-list enumerables and a `{:badmap, _}` error for non-maps; total over the input shapes their `@spec` and docs describe. Basis: `asserted` (all clauses present and ordered map-list → map-nonlist → non-map).
- **`equal?/2` uses `===`** — doctest `Map.equal?(%{a: 1.0}, %{a: 1})` → `false` is consistent with `map1 === map2`. Basis: `asserted` (direct correspondence between code and doctest).
- **`merge/3` / `intersect/3` delegation** — delegate to `:maps.merge_with`/`:maps.intersect_with` with the arg order `(fun, map1, map2)`, matching the documented "value2 has precedence"/conflict-function semantics. Verified `Map.intersect(%{a: 1, b: 2}, %{b: "b", c: "c"})` → `%{b: "b"}`. Basis: `verified`.

## Risks / caveats
- Review covered only `/Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/map.ex`; the delegated Erlang `:maps` primitives (`from_list`, `take`, `update`, `merge_with`, `intersect_with`, iterators) are out of scope and assumed correct.
- The non-`iex>` examples in `@doc` for `keys/1`, `values/1`, and the `fetch!`/`replace!` KeyError snippets are intentionally not executable doctests (map key order is unspecified), so they cannot be validated by running and were not treated as contract violations.
- `is_map`-guarded functions (`replace_lazy/3`, `filter/2`, `reject/2`, `split_with/2`) raise `FunctionClauseError` rather than `{:badmap, _}` on non-map input, a minor inconsistency with the `case`-based functions, but this is only reachable by violating the `@spec` (non-map argument) and is therefore out of scope per the review rules.

