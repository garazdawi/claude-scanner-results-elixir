# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_for.erl
<!-- sha256: b5367cdcd8e7f3fc7bff244ea7384a3e8325fdd16a89e26731604aa28d9d2fae -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_for.erl
<!-- sha256: b5367cdcd8e7f3fc7bff244ea7384a3e8325fdd16a89e26731604aa28d9d2fae -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T (hedge-resolution pass)_

# Bug-hunting review — `elixir_erl_for.erl`

## Summary
The module correctly compiles `for` comprehensions to Erlang abstract format; the only concrete defect found is an unreachable optimization clause, and everything else in the failure-mode inventory checks out. In this re-review I read the three in-folder callees that the first pass hedged on (`elixir_erl_clauses:guards/4`/`match/4`, `elixir_erl_var:build/2`, `elixir_utils:split_last/1`/`extract_guards/1`/`returns_boolean/1`). Both hedged assumptions are now verified against those modules; neither becomes a finding.

## Bugfixes

#### `CSCAN-elixir_erl_for-map-into-optim-dead`
- **Finding ID** — `CSCAN-elixir_erl_for-map-into-optim-dead`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `elixir_erl_for.erl:134-138`, `build_inline_each/7` (the `{map, _, []} = _Into` clause)
- **Class** — Dead / unreachable code
- **User-visible outcome** — For a simple `for x <- enum, into: %{}, do: {k, v}` (single-variable generator, no filters, non-uniq), the intended fast path — `maps:from_list(Enum.map(enum, Fun))` — never fires. Instead the code takes the general path (`maps:from_list(lists:reverse(Enum.reduce(...)))`), so the generated code builds a list via `reduce`, reverses it, then converts. The result is *correct*; the caller observes only a slightly larger/slower generated function than the author intended for this common case. There is no wrong output a caller would see — the incorrect behaviour is strictly a missed optimization (extra `lists:reverse` + `reduce`-with-cons instead of a direct `Enum.map`).
- **Reproduction** — Trace `translate/3` for `for x <- list, into: %{}, do: {x, x}`. `TInto` is `{map, Ann, []}`; `inline_or_into/1` (line 353-356) returns `into` for a non-empty-pattern map (falls to catch-all `_ -> into`), so `translate_into` calls `build_into/7`. `build_into`'s first clause (line 164) matches `{map, _, []}` and re-dispatches to `build_inline_each(..., {nil, Ann}, ...)` — it passes `{nil, Ann}`, never `{map, _, []}`. The only other caller of `build_inline_each/7` is `build_inline/7` (line 124), reached only when `inline_or_into` returned `inline`, i.e. Into ∈ `{false, {nil,_}, {bin,_,[]}}`. No caller ever passes `{map, _, []}`, so the clause at 134 is unreachable.
- **Suggested fix** — Either delete the `{map, _, []}` clause (134-138) as dead, or, to actually enable the optimization, change `build_into/7`'s empty-map clause (164-166) to pass the map through so the specialized clause can match. Deletion is the minimal, behaviour-preserving change.
- **Risk of fix** — Deleting the clause: none (unreachable). Enabling the optimization: changes the emitted abstract form for `into: %{}` with a simple generator; anything asserting on generated Core/BEAM shape would shift, but runtime semantics are unchanged.
- **Evidence** — read `elixir_erl_for.erl:52-55` (dispatch), `:121-166` (`build_inline`/`build_inline_each`/`build_into`), `:349-356` (`inline_or_into`). Confirmed unchanged in re-review.

## Risks / caveats
- No `-spec` exists on the sole export `translate/3`; spec-totality could not be checked against a declared type. `translate/3` requires `Args` to end with `[{do, Expr} | Opts]`. Verified against `elixir_utils:split_last/1` (`elixir_utils.erl:80-83`): it returns `{AllButLast, LastElem}`, so line 15's match `{Cases, [{do, Expr} | Opts]}` `badmatch`es only if the final arg is not a `[{do,...}|_]` list — a caller-contract violation guaranteed away by `elixir_expand`, out of scope.
- The first pass hedged that the guard-list returned by `elixir_erl_clauses:guards/4` might carry an inner list with ≥2 elements, which would `function_clause` in `translate_guards/1` (line 95-100). **Now verified by reading `elixir_erl_clauses.erl:40-48`:** `guards(Ann, Guards, Extra, S)` is called from `elixir_erl_for.erl:80` with `Extra = []`. For each guard group it returns `translate_guard(Guard, Ann, SG, [])` = `[element(1, translate(Guard,...)) | []]`, i.e. an inner list of **exactly length 1**; when `Guards == []` and `Extra == []` it returns `[]`. So the value threaded into `translate_guards/1` is always shaped `[]` or `[[E1], [E2], ...]` with every inner list a singleton — precisely the three clauses `translate_guards/1` covers (`[]`, `(uncheckable memory citation: G)`, `[[L],[R]|Rest]`). No `function_clause` is reachable. This dependency is now **verified**, not assumed.

## Verified callee-shape dependencies (were hedged in first pass)
- `elixir_erl_pass:translate/3` — used pervasively; every call site destructures `{TExpr, S}`. Consistent with all in-folder callers (`elixir_erl_clauses.erl:24,35,48,64-65` all treat it as returning a `{Form, Scope}` pair). Contract shape confirmed by neighbour usage.
- `elixir_erl_clauses:match/4` (`elixir_erl_clauses.erl:23-27`) — returns `{Result, NewS}`, restoring the caller's `context`. Matches the `{TLeft, SL}` destructure at `elixir_erl_for.erl:77`. **Verified.**
- `elixir_erl_clauses:guards/4` (`elixir_erl_clauses.erl:40-48`) — return shape `[[SingleExpr], ...]`, as detailed above. **Verified.**
- `elixir_erl_var:build/2` (`elixir_erl_var.erl:39-46`) — returns `{AtomName, UpdatedScope}` (fresh `_N` atom + bumped counter). Matches `build_var/2`'s wrap into `{{var, Ann, Name}, ST}` at `elixir_erl_for.erl:309-311`. **Verified.**
- `elixir_utils:split_last/1` (`elixir_utils.erl:80-83`), `extract_guards/1` (`:61-65`, returns `{Left, FlatGuardList}`), `returns_boolean/1` (`:207-248`, always boolean) — all shapes match their use at `elixir_erl_for.erl:15,76,104`. **Verified.**

### Ruled out

**Operator / precedence typo**
- Reason — `not Uniq and lists:all(...)` (line 122) parses as `(not Uniq) and (...)` correctly, and the `orelse`/`==` combinations in `join_filter` (375-378) match the intended false/nil short-circuit.
- Axes examined: uniq true/false, all-bin vs mixed clauses, filter var false/nil | Not examined: none
- Basis — verified

**Missing clause / catch-all**
- Reason — every `Into` shape reaching `build_inline_each`/`build_into`/`comprehension_kind`/`inline_or_into` is covered, and the generator dispatch in `translate_gen`/`build_reduce_each` handles `enum`, `bin`, and empty tails. Cross-checked against `translate_guards/1`: the guard-list shape from `elixir_erl_clauses:guards/4` (inner lists always length 1) is fully covered by its three clauses.
- Axes examined: Into = false/{nil}/{bin,[]}/{map,[]}/MapSet/other; clauses empty, single, mixed enum+bin; guard-groups 0/1/≥2 | Not examined: none
- Basis — verified

**State / accumulator threading error**
- Reason — scope vars thread monotonically (`S→SI→SC→SE`, `SF→SA→SK→SR→ST→SD→SN`); the discarded inner scopes in `build_reduce` are intentional (comment line 215-216) since those vars live only inside generated funs bound in their own clauses. `elixir_erl_var:build/2` confirmed to return an updated scope with a bumped counter, so the discards drop only counter state that is never observed downstream.
- Axes examined: reduce/into/inline paths, uniq true/false, nested generators | Not examined: none
- Basis — verified

**Off-by-one / wrong arithmetic**
- Reason — no numeric index/length arithmetic; `element(1|2|3, ...)` index fixed tuple positions (result tuple, annotation, fun name) that are structurally correct.
- Axes examined: element index on tuple/op/fun forms | Not examined: none
- Basis — verified

**Contract violation**
- Reason — `translate/3` always returns `{AbstractForm, Scope}`; every helper returns the `{Expr, S}` pair its callers destructure. Confirmed the two upstream callee contracts (`match/4`, `guards/4`) supply the shapes assumed.
- Axes examined: all return sites of translate/build_* | Not examined: none
- Basis — verified

**Crash on stray / unexpected input**
- Reason — inputs are post-expansion Elixir AST guaranteed by `elixir_expand`/`elixir_erl_pass`; the one destructuring `badmatch` (line 15) requires a caller-contract violation. `split_last/1` shape (verified) means the match can only fail when the last arg is not `[{do,_}|_]`, which is contract-illegal.
- Axes examined: malformed Opts, missing do, non-list Args | Not examined: none (all require contract-illegal input)
- Basis — verified

**Resource leak on error path** / **Race condition / shared mutable state** / **Infinite loop / hang**
- Reason — pure AST-to-AST translation; no files, ETS, processes, monitors, receives, or shared state; all recursion is structural over finite clause/element lists.
- Axes examined: recursion over clauses/filters/bin-elements | Not examined: none
- Basis — verified

**Silent corruption**
- Reason — filter/guard nesting order is correct: `collect_filters` reverses filters, `mapfoldr` preserves that order, and `join_filters` folds so outer=first-evaluated (guard/extra before filters), matching source order.
- Axes examined: 0/1/2 filters, pattern guard present/absent, extra pin guards | Not examined: none
- Basis — verified

**Quadratic-where-linear / Repeated work**
- Reason — list building uses `[H|Acc]` + `lists:reverse`, and comprehensions map/fold once; no `++` in a hot recursive accumulator.
- Axes examined: accumulator construction in translate_gen/collect_filters/join_filters | Not examined: none
- Basis — verified

**Documented behaviour mismatch**
- Reason — no `@doc`/`-spec`/prose contract in the file to contradict; behaviour matches the implicit `for`-compilation contract exercised by the two callers.
- Axes examined: module has no doc/spec prose | Not examined: none
- Basis — verified

**Wrong scope / order of operations**
- Reason — `build_into` builds the `Collectable.into` match before the reduce and the `done`/`halt` handling in the correct order (match → try reduce → done, stacktrace → halt → raise).
- Axes examined: into match/try/stacktrace ordering | Not examined: none
- Basis — verified

**Encoding / unicode mishandling**
- Reason — no byte/codepoint handling; `no_var_expr` correctly expands a `{string,_,S}` bin literal to one `_` element per element of `S`.
- Axes examined: string bin element vs non-string | Not examined: none
- Basis — verified

**Unclassified**
- Reason — the one genuine issue (unreachable map-into clause) is reported above under Dead code; no other cross-shape anomaly found. Line 336 `_vars` vs `_Vars` is a cosmetic mismatched-but-both-discarded catch-all argument, not a bug.
- Axes examined: full read of all 384 lines | Not examined: none
- Basis — verified

### Did I verify this?
Yes. In the first pass the sole `Depends on` residue was the guards-shape assumption for `elixir_erl_clauses:guards/4` plus the general return-shape assumptions for `elixir_erl_pass:translate/3`, `elixir_erl_var:build/2`, and `elixir_utils:split_last/1`/`extract_guards/1`/`returns_boolean/1`. In this re-review I read `elixir_erl_clauses.erl` (`match/4:23-27`, `guards/4:40-48`), `elixir_erl_var.erl` (`build/2:39-46`), and the relevant `elixir_utils.erl` definitions (`split_last:80-83`, `extract_guards:61-65`, `returns_boolean:207-248`). Every hedged shape is confirmed, so the `translate_guards/1` no-`function_clause` conclusion and the contract-shape ruleouts are now `verified` rather than assumed. All ruled-out entries remain Basis `verified` with `Not examined: none`; those entries were closed by reading this file's 384 lines and enumerating the datatype's Into/clause/filter/guard-group axes, and none of their `Not examined` lists were narrowed by neighbour reads (they were already empty).

### Dropped in re-review
None. The single finding (`CSCAN-elixir_erl_for-map-into-optim-dead`) is preserved; re-tracing the dispatch confirmed the `{map,_,[]}` clause at line 134 remains unreachable. No prior finding was dropped, and no hedged assumption had to be upgraded to a low-confidence finding because both were verified directly against the named neighbour modules.
