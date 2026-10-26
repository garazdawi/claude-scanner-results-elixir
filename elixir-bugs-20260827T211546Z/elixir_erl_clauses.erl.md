# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_clauses.erl
<!-- sha256: e4a16fa7974923e6bcfbe3078df97a5478776facf10e5286cdc790ae4a096630 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T06:08:16Z_

# Review: `elixir_erl_clauses.erl`

## Summary
This 70-line Erlang module (translation of case/fn/receive clauses, guards, and matches into Erlang abstract format) is a faithful, internally-consistent implementation with no reachable defects found.

## Bugfixes

No findings.

The module has no `-spec` declarations, so the spec-totality checklist is vacuously satisfied (no exported function under-declares its spec because none carry one). Walking each export:

- `get_clauses/3` — `lists:keyfind` with an `is_list` guard on the value and a `_ -> []` catch-all; the comprehension only binds `{'->', Meta, [Left, Right]}` shapes and silently skips others (generator filtering, not a crash). Total over its inputs.
- `match/4` — two clauses fully partition on `context =/= match` vs. otherwise; context is saved and restored correctly (`NewS#elixir_erl{context=Context}`), and `Fun(Match, Ann, S)` argument order matches `elixir_erl_pass:translate/3` / `translate_args/3`.
- `clause/6` — state threading is correct: `SA` (post-match) supplies `extra_guards` to `guards/4`, `SG` resets `extra_guards=[]` and feeds the expr translation, and `SE` (post-expr) is returned. The guard-translation state is intentionally discarded — Erlang guards bind no escaping variables — consistent with upstream.
- `guards/4` — the `[]`/non-`[]` split and the nested `Extra` empty check correctly emit `[]`, `[Extra]`, or the translated guard list.
- `clauses/2` — `[]` short-circuit plus `lists:mapfoldl(each_clause, ...)`; `each_clause/2`’s `{match,...}`/`{expr,...}` heads match exactly the tuple shapes the in-folder callers (`elixir_erl`, `elixir_erl_for`, `elixir_erl_pass`, `elixir_erl_try`) construct.
- `unblock/1` — `{'block',_,Exprs}` unwrap vs. `[Exprs]` wrap is total.

## Risks / caveats
- `each_clause/2` will `function_clause` on any clause tuple that is neither `{match,...}` nor `{expr,...}`, and `clauses/2`/`clause/6` assume the four-element `{As,Meta,Left,Right}` shape produced by `get_clauses/3`. These are compiler-internal, contract-bound inputs (not reachable with attacker-malformed shapes through a documented public API), so they are not findings — but a future caller emitting a new clause kind would crash here rather than degrade gracefully.
- All dismissals below depend on the behaviour of the neighbour modules named in the callgraph slice (`elixir_erl_pass`, `elixir_utils`) matching their documented 3-arity `translate`/`translate_args` and `extract_guards` contracts; if `elixir_erl_pass:translate/3` were changed to return a shape other than `{Abstract, State}`, `clause/6` and `each_clause/2`’s `{T, S}` destructuring would `badmatch`. That is an out-of-file invariant.

### Ruled out

**Operator / precedence typo**\
Reason: the only compound conditionals are the nested `case Extra of [] -> ...` in `guards/4` and the single `Context =/= match` guard in `match/4`; neither uses `,`/`;` guard sequencing or boolean `andalso`/`orelse`, so there is no precedence to mis-order.\
Axes examined: empty guard list, non-empty guard list, empty extra-guards, non-empty extra-guards, context=match vs other | Not examined: none\
Basis: verified

**Missing clause / catch-all**\
Reason: `get_clauses/3` has a `_ ->` fallthrough; `match/4`, `guards/4`, `unblock/1` are total over their declared domains; the only non-total heads (`each_clause/2`, `clauses/2` recursion) take compiler-constructed tuples, not public-API input.\
Axes examined: keyfind hit/miss, non-list value, `{match}` vs `{expr}` clause, block vs non-block expr | Not examined: clause kinds other than match/expr (internal-only, out of scope)\
Basis: verified

**State / accumulator threading error**\
Reason: `clause/6` correctly uses `SA` for extra_guards, `SG` (reset) for expr translation, and returns `SE`; `match/4` restores the caller’s context; `clauses/2` uses `mapfoldl` threading state across clauses. The discarded guard-state is deliberate (guards bind nothing that escapes).\
Axes examined: context save/restore, extra_guards propagation, mapfoldl state threading, single vs multiple clauses | Not examined: none\
Basis: verified

**Contract violation / documented behaviour mismatch**\
Reason: no `-spec` or docstring makes a promise the code breaks; return shapes (`{clause,...}` tuples and `{Translated, State}` pairs) are consistent across `clause/6`, `each_clause/2`, `clauses/2`.\
Axes examined: return-tuple arity, empty-clause-list return `{[],S}`, comprehension output shape | Not examined: none\
Basis: verified

**Crash on stray / unexpected input**\
Reason: the only public-API surface reachable with free-form data is `get_clauses/3`’s `Keyword`, which is fully guarded (`is_list`, catch-all, generator filtering); other entry points consume compiler-internal AST whose shape is a caller contract.\
Axes examined: keyword missing key, non-list clauses value, malformed `->` entries, empty keyword | Not examined: none\
Basis: verified

**Quadratic-where-linear / Repeated work**\
Reason: no accumulator uses `++`; the two list comprehensions and `mapfoldl` are single-pass; no pattern compilation or repeated scans occur.\
Axes examined: list-comprehension building, mapfoldl accumulation | Not examined: none\
Basis: verified

**Resource leak / Infinite loop / Race condition**\
Reason: the module is pure list/tuple transformation — no files, ETS, processes, monitors, receives, or shared mutable state are touched.\
Axes examined: full read for side-effecting BIFs (none present) | Not examined: none\
Basis: verified

**Off-by-one / Encoding / Wrong atom / Dead code / Wrong scope**\
Reason: no arithmetic, no byte/codepoint handling, no atom-name literals beyond `'->'`/`'block'`/`match`/`guard` (all correct and matched consistently), no unreachable branches (every clause is reached by a distinct input shape), and no side-effect ordering.\
Axes examined: atom literals cross-checked against usage, block-unwrap tag, context atoms | Not examined: none\
Basis: verified

### Did I verify this?

Every ruled-out entry above is `verified`: the file is 70 lines and was read in full, and for each class I enumerated the datatype axes (keyword shapes, clause-kind tuples, guard/extra-guard emptiness combinations, block vs non-block expressions, context values) and traced each through the relevant clause. The only untried axes are clause-kind tuples other than `{match,...}`/`{expr,...}`, which are internal compiler constructions outside the documented public-API scope and are explicitly noted as such rather than dismissed. No entry rests on an `asserted` basis, so no `Depends on:` lines remain — except the standing cross-file assumption (stated under Risks / caveats) that `elixir_erl_pass` and `elixir_utils` honour their 3-arity translation and `extract_guards` return contracts, which is an out-of-file invariant the cross-file pass must confirm.

