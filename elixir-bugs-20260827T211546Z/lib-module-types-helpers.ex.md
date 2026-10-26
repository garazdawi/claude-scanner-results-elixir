# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/helpers.ex
<!-- sha256: a7d911c43fdf9ef0d816ef4160c9043ff361ea6dba8a167f1b18d66f6ca374f8 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:52:14Z_

## Summary

`Module.Types.Helpers` (a `@moduledoc false` compiler-internal helper module) is clean: every exported function I exercised — ordinal formatting, list/match unpacking, trace collection, the `&&`/`||`/`and`/`or`/`if` case-rewrites in `expr_to_string`, struct collapsing, and hint formatting — behaves per its documentation, and the only crashes I could produce require callers to violate the documented contract.

## Bugfixes

No findings

## Risks / caveats

- The module is `@moduledoc false`; strictly there is no public API. I treated the exported functions as the callable surface (as the compiler's own type-checking pipeline calls them) and hunted for bugs reachable that way.
- I could not construct a genuine `collect_traces`/`format_traces` payload without a valid `Module.Types.Descr` value; my first attempt failed inside `Descr` on a malformed type I supplied (my test error, not a defect). The verified run in `t3.exs` used real `Descr.integer()` values.
- I did not trace every external call site of `format_hints/1` across the whole codebase; my "unknown hint crashes are a contract" reasoning rests on `expr_hints/1` (the in-file producer) only ever yielding handled atoms.

### Ruled out

- **`Macro.prewalk` fun in `collect_traces` returns `{:ok, versions}` as the `{node, acc}` pair, replacing matched variable nodes with the atom `:ok`.** Not a bug: variable nodes are leaves, so replacing them never blocks descent into sibling/child subtrees, and the rewritten AST is discarded (`{_, versions} = ...`). Verified in `t3.exs`: an expr with two distinct versioned vars yields both traces (`names: [:x, :y]`). Basis: `verified`.
- **All operator reconstructions in `expr_to_string` (`&&`, `||`, `and`, `or`, `!`, both `if` shapes) invert the compiled `case` correctly.** Verified in `t4.exs`: `and`→`"a and b"`, `or`→`"a or b"`, `&&`→`"a && b"`, `if`→`"if a do dob else elseb end"`, each matching the truthiness semantics of the source construct. Basis: `verified`.
- **`integer_to_ordinal/1` teen-suffix logic (11/12/13 → "th").** Verified in `t.exs`: `1→"1st"`, `11→"11th"`, `21→"21st"`, `112→"112th"`, `0→"0th"`. Negative inputs would yield e.g. `"-1th"`, but the function names ordinal argument positions (always ≥ 1), so negatives are unreachable. Basis: `verified`.
- **Struct collapsing / `:collapse_structs` option in `expr_to_string`.** Verified in `t5.exs`: default collapses only fields matching struct defaults (leaving `...`), and `collapse_structs: false` prints fields verbatim — matching the `@doc`. Basis: `verified`.
- **`unpack_list([], _)` and `zip_map_reduce/4` on unequal-length lists raise `FunctionClauseError`.** Verified in `t5.exs`. These are contract violations (a list to unpack always has ≥1 element in the parser's usage; the two zip lists are arg lists of equal arity), so raising is the intended behavior, not a bug. Basis: `verified`.
- **`is_var/1` evaluates its argument five times.** It expands to a guard-safe boolean (`is_tuple ... and tuple_size ... and is_atom(elem ...)`); it is documented as a guard function and used in guard position, where re-evaluation is side-effect-free and `and` short-circuits before `elem/2` on non-tuples. No double-side-effect path exists. Basis: `asserted` (guard semantics), corroborated by the guard-safe expansion in `t4.exs`'s successful use of the same AST shapes.
- **`format_hints/1` has no catch-all clause.** The only in-file producer, `expr_hints/1`, yields exclusively handled atoms (`:inferred_bitstring_spec`, `:anonymous_rescue`, `[]`); all other in-repo call sites pass literal handled atoms (`:dot`, `:empty_domain`, `{:impl, _}`, etc.), all of which I confirmed produce output in `t.exs`. An unknown atom would raise, which is the intended closed-set contract. Basis: `verified` for handled atoms; `asserted` for the closed-set claim.

