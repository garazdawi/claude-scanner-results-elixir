# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_pass.erl
<!-- sha256: 1c57ce37f0a32c869fccdbd9e04331d316936adb831e458482141117ce961218 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T06:11:53Z_

# Bug-hunting review — `elixir_erl_pass.erl`

## Summary
After tracing every documented export (`translate/3`, `translate_args/3`, `no_parens_remote/2`, `parens_map_field/2`) and the Erlang-specific rewrite/optimization paths, I found no correctness, crash, or contract defect reachable through the module's public API.

## Bugfixes

No findings.

The high-value paths I scrutinized and cleared:

- **`lists:member` optimization state threading (lines 583–620).** `translate(Head/X, Ann, S)` at lines 603/608 deliberately discard the returned state and reuse the original `S`. This is safe because `optimize_list_membership/2` (lines 695–701) admits the branch only when every list element is `is_number`/`is_atom`/`is_binary`; translating such literals allocates no variables, so the discarded states carry no counter drift. `S3` (derived from the single `TExpr` translation) is the state actually returned. The emitted `=:=` chain matches `lists:member/2`'s exact-equality semantics (`1 =:= 1.0` → false, consistent with `lists:member`).
- **`maps:merge` literal fusion (lines 621–636).** All four clauses preserve Map2-wins semantics (`Pairs1 ++ Pairs2` puts Map2 last, and Erlang map literals let the later `=>` win) and preserve left-to-right evaluation order.
- **`evaluate_first/4` (lines 766–773) + reorder table.** The fast path fires only when the leading arg is a variable node (`is_atom(Var), is_atom(Ctx)`), which is side-effect-free, so the argument reordering (`elem`→`element`, `put`→`maps:put`, `duplicate`→`make_tuple`, etc.) never reorders an effectful first argument; otherwise the first arg is pre-bound via a `match` block. Relative order of the remaining args is preserved by every `?reorder` closure.
- **`inline_pure`/`inline_args` (lines 680–693, 703–733).** `apply(Left, Right, Args)` runs at compile time only when `basic_type_arg/1` (numbers/atoms/binaries/2-tuples/lists thereof) holds, and the resulting raw term is re-fed through `translate/3`, whose literal fall-throughs (lines 293–302) correctly turn maps/tuples/lists/scalars back into abstract format. `shift` inlining is guarded by `basic_type_arg(Opts)` and a `try/catch _:_` that falls back to `none`.
- **`no_parens_remote/2` (lines 815–831).** Clause set is total over any term (`nil`/`false`/`true` → `badmap`, `is_atom` → apply, `#{}` → `badkey`, `Other` → `badmap`).

## Risks / caveats
- `translate/3` operates on already macro-expanded AST; malformed node shapes (e.g. a remote-call node whose function position is a non-atom) would fall through to `elixir_erl:elixir_to_erl/2` or fail to match, but producing such shapes is a contract violation by the expander, not reachable via well-formed public use — excluded per scope.
- Rulings that discard translator state on the `lists:member`/reorder literal paths depend on `elixir_erl_var:build/assign` and `elixir_erl:elixir_to_erl/2` not mutating counters for pure literals; this matches those neighbours' documented behaviour but was not re-verified inside their source in this single-file pass.

### Ruled out

- **Operator / precedence typo**\
  Reason: The `if`-guards and boolean chains (lines 88–94, 134, 391–394, 641, 696) use the intended connectives; `/=` vs `=/=` at line 134/391 is immaterial because the operands are atoms.\
  Axes examined: atom conditions, integer/float exact-vs-coerced equality, guard `;`/`,` partitions | Not examined: none\
  Basis: verified

- **Missing clause / catch-all**\
  Reason: `translate/3` ends in an `Other` fall-through (line 301); the `if` in `super` and `case` dispatches cover the shapes the expander emits; literal/term paths are total.\
  Axes examined: scalar, 2-tuple, 3-tuple non-remote, list, map, empty list | Not examined: none\
  Basis: verified

- **Wrong atom / wrong symbol**\
  Reason: BIF/module/atom names in the reorder table and rewrites (`erlang:element`, `maps:put`, `map_field_exact`/`map_field_assoc`, `__duration__!`, `__to_existing_atom__`) are consistent with their Erlang targets and Elixir field names.\
  Axes examined: reorder targets, map-field record tags, struct key `__struct__` | Not examined: none\
  Basis: verified

- **State / accumulator threading error**\
  Reason: `St0/St1` chains verified in `=`, `translate_map`, `translate_with_do`, and the `lists:member`/`evaluate_first` optimizations; discarded states are provably empty-of-effects for the literal paths.\
  Axes examined: var-first vs expr-first args, literal-only lists, nested map key/value extra flags | Not examined: none\
  Basis: verified

- **Off-by-one / wrong arithmetic**\
  Reason: `increment/2` (lines 763–764) correctly maps Elixir 0-based `elem`/`put_elem`/`insert_at`/`delete_at` to Erlang 1-based `element`/`setelement`/… ; `optimize_list_membership` bound (≤33 elements) is a cap, not a boundary error.\
  Axes examined: literal integer index, non-literal index, list length 0/1/33/34 | Not examined: none\
  Basis: verified

- **Contract violation**\
  Reason: Every export returns the documented `{ErlAST, State}` (or, for `no_parens_remote`, `{ok,_}`/`{error,_}`) shape across all clauses.\
  Axes examined: each export's clause set | Not examined: none\
  Basis: verified

- **Crash on stray / unexpected input**\
  Reason: Crashes only occur on AST shapes the expander cannot emit (contract violations, out of scope); literal fall-throughs absorb arbitrary terms.\
  Axes examined: raw scalars/maps/tuples/lists through `translate`, `no_parens_remote` on all term kinds | Not examined: none\
  Basis: verified

- **Resource leak on error path**\
  Reason: No files, ETS tables, processes, or monitors are created in this module; only pure AST transformation. `try/catch _:_` blocks (lines 713, 725) fall back to `none` without acquiring resources.\
  Axes examined: try/catch fallbacks | Not examined: none\
  Basis: verified

- **Silent corruption**\
  Reason: `maps:merge`/`lists:member`/`to_existing_atom`/`to_string` rewrites preserve observable semantics and evaluation order as traced above.\
  Axes examined: duplicate map keys, exact-equality membership, effectful-arg ordering | Not examined: none\
  Basis: verified

- **Infinite loop / hang**\
  Reason: All recursion (`translate_list`, `build_list`, `translate_block`, `build_cond_clauses`, `extract_bit_type`) decreases on a shrinking list/tree; no `receive`/blocking constructs.\
  Axes examined: empty/single/multi-element lists, nested blocks | Not examined: none\
  Basis: verified

- **Quadratic-where-linear**\
  Reason: `Pairs1 ++ Pairs2` (line 626) is a one-shot fusion of two literal maps, not a loop; accumulators use `[H|Acc]` + `lists:reverse` (lines 69, 515, 808). `Do ++ After` (line 180) and `Else ++ [RaiseClause]` (line 437) are single small concatenations.\
  Axes examined: accumulator idioms, per-call concatenations | Not examined: none\
  Basis: verified

- **Repeated work**\
  Reason: No pattern compilation, BEAM scanning, or repeated expensive calls; each node is translated once.\
  Axes examined: translate call counts per node | Not examined: none\
  Basis: verified

- **Dead / unreachable code**\
  Reason: The `extract_bit_type({Other,_,[]}, ...)` clause (line 558) is documented legacy support for pre-1.14 debug_info and is intentionally retained; all other clauses are reachable from documented AST shapes.\
  Axes examined: legacy bit-type list form, reorder table entries | Not examined: none\
  Basis: verified

- **Documented behaviour mismatch**\
  Reason: Deprecation warnings and return values in `no_parens_remote/2`/`parens_map_field/2` match their doc comments; rewrite optimizations preserve the semantics of the functions they replace.\
  Axes examined: `no_parens_remote` return contract, warn_once behaviour | Not examined: none\
  Basis: verified

- **Race condition / shared mutable state**\
  Reason: Pure functional translation with no shared/process state.\
  Axes examined: module-level state usage | Not examined: none\
  Basis: verified

- **Wrong scope / order of operations**\
  Reason: `evaluate_first` and the reorder closures preserve Elixir left-to-right evaluation order even after argument permutation.\
  Axes examined: var-first vs expr-first, effectful non-first args | Not examined: none\
  Basis: verified

- **Encoding / unicode mishandling**\
  Reason: Binary bitstring literals use `binary_to_list/1` only for non-UTF types and `elixir_utils:characters_to_list/1` for UTF types (lines 517–527), matching `elixir_erl:elixir_to_erl`'s documented rationale.\
  Axes examined: utf vs binary/bitstring bit types | Not examined: none\
  Basis: verified

### Did I verify this?

Every ruled-out class above is marked `verified`: for each I read the governing clauses in this file and traced the triggering inputs across the datatype's axes (empty/single/multi lists, literal vs non-literal args, var-first vs expr-first, scalar/tuple/map/list terms, exact vs coerced equality, legacy bit-type forms), leaving each `Not examined` list at `none`. The two dependencies I could not re-verify inside their own source — `elixir_erl_var` counter behaviour for pure literals and `elixir_erl:elixir_to_erl/2` term handling — are recorded in Risks / caveats rather than used to close a would-be finding; they do not, on any assumption I could construct, upgrade a ruled-out class to a finding, because the optimization paths that rely on them operate exclusively on side-effect-free literal terms.

