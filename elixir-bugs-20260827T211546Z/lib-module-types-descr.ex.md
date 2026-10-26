# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/descr.ex
<!-- sha256: 1057797db27539ddf6ae0f4ffcf27371643a0ac8322318e3c4cee97b9bf306ad -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:50:11Z_

# Summary

Extensive differential fuzzing (opt vs. bare set operations, optimized predicates vs. brute force, accessor crash-fuzzing, and semantic models for truthiness/fun_apply/tuple-list-map accessors) found the operational core of this module to be correct; the one defect is a documentation-contract mismatch in `compatible_intersection/2`.

# Bugfixes

#### `CSCAN-descr-compatible-intersection-error-shape`
- **Finding ID** — `CSCAN-descr-compatible-intersection-error-shape`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/elixir/lib/module/types/descr.ex:955` (`@doc` for `compatible_intersection/2`, defined at lines 961 and 969)
- **Issue** — The `@doc` states: *"Returns the intersection between two types only if they are compatible. Otherwise returns `:error`."* Every code path actually returns a two-element tuple — `{:ok, descr}` on success and `{:error, descr}` on incompatibility (lines 962–966, 981, 985, 988) — never the bare atom `:error`.
- **Class** — Documentation mismatch
- **Impact** — A caller who follows the documentation and writes `case compatible_intersection(a, b) do :error -> ... end` (or `{:ok, t} -> ... ; :error -> ...`) never matches the incompatible branch; the `{:error, _}` tuple falls through, producing a `CaseClauseError` or silently taking the wrong branch. The sole in-tree caller (`of.ex:140`) happens to use a catch-all `_`, so no current failure occurs, which is why the divergent doc has gone unnoticed — but it misleads every future consumer of this compiler-internal API.
- **Evidence** — `verified`. Command: `compatible_intersection(integer(), atom())` → `{:error, %{bitmap: 8}}`; `compatible_intersection(none(), term())` → `{:error, %{}}`; `compatible_intersection(integer(), term())` → `{:ok, %{bitmap: 8}}`. In 40,000 random pairs, `compatible?(a,b)` matched `match?({:ok, _}, compatible_intersection(a,b))` exactly (0 mismatches), confirming `{:ok,_}`/`{:error,_}` is the real, intended contract and the `:error` in the doc is wrong.
- **Suggested fix** — Change line 955 to describe the real shape, e.g. *"Returns `{:ok, intersection}` if the types are compatible, otherwise `{:error, left}`."*

# Risks / caveats

- This is a ~7200-line set-theoretic type engine. I verified the documented public surface behaviorally rather than by exhaustively proving each internal BDD/phi/map-line algorithm; a subtle soundness bug in a code path my corpus never constructed (e.g. deeply-nested recursive equations via `recursive/1`, or specific negation shapes in `phi`/`map_line_empty?`) could survive this review. The invariants I did check held on every generated case.
- `numberize/1` (line 568) recurses into `:tuple`, `:map`, `:list`, and `:dynamic` but not `:fun`, and within maps it rewrites field values but not domain-key value types. This may be intentional (function types are weak), but "Converts all floats or integers into numbers" (line 565) slightly overstates coverage; I did not classify it as a bug because no documented caller contract is violated.

### Ruled out
- **`opt_union`/`opt_intersection`/`opt_difference`/`opt_negation` diverging from `bare_*`** — 40,000 random pairs over a corpus including nested tuples/maps/lists, negations, and dynamic wrappers; `equal?` held in every case. Basis: `verified`.
- **`empty_list_type?`/`bitstring_type?`/`bitstring_no_binary_type?`/`number_type?` diverging from their documented brute-force `not empty?(intersection(...))` definitions** — 374-type corpus, 0 mismatches each. Basis: `verified`.
- **`truthiness/1` violating its `:undefined`/`:always_true`/`:always_false` contract** — matched an independent subtype/disjoint model on a targeted atom-heavy corpus (0 mismatches). Basis: `verified`.
- **`tuple_insert_at`/`tuple_delete_at`/`tuple_replace_at` off-by-one / wrong-index** — concrete closed- and open-tuple models (`{a,b}`, `{a,b,c}`, `{a,b,...}`) matched expected results including out-of-range `:badindex`. Basis: `verified`.
- **`fun_apply` returning wrong result type for non-overlapping and inferred/overlapping clauses, static and dynamic** — results matched hand-computed expectations (`:foo`/`:bar`/union, `:badarg` domain = `number()`). Basis: `verified`.
- **`compatible?/2` inconsistent with `compatible_intersection/2`** — after correcting the predicate to `{:ok, _}`, 40,000 random pairs agreed exactly; the two are consistent (only the doc string is wrong). Basis: `verified`.
- **`to_quoted_string/2` crashing on valid types** — 30,000 random pairs including unions/intersections/differences of nested/negated/dynamic types produced no crash. Basis: `verified`.
- **Documented map/tuple/list accessors (`map_get`, `map_fetch_key`, `map_put`, `map_update`, `map_to_list`, `tuple_fetch`, `tuple_values`, `list_hd`, `list_tl`, `list_of`) crashing on valid input** — 40,000 randomized calls, 0 crashes; `list_hd`/`list_tl`/`list_of` also matched their documented tail/head semantics on concrete list types. Basis: `verified`.

