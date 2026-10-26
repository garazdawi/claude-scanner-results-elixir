# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types.ex
<!-- sha256: 0b46de6955ab6da422d482881ab5052d8c391f3838d83e9531a727addf01d3e4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:46:32Z_

The format functions all behave correctly (I initially misread the output order — count=2/arity=3 → `arity-count==1` → singular clause B; count=1/arity=3 → clause C "last 2"; both correct).

After a thorough pass over the whole file, here is my report.

# Review: `lib/module/types.ex` (`Module.Types`)

## Summary
This compiler-internal type-inference driver (`@moduledoc false`, every function `@doc false`) threads its accumulators, mode dispatch, error-tuple formatting, and default-argument reachability logic consistently, and I found no reachable correctness defect.

## Bugfixes

No findings

## Ruled out

- **`format_error/1` default-argument plurality (lines 731–741)** — I suspected the `arity - count == 1` guard on the middle clause could route the singular/plural messages to the wrong count. Driven directly: `format_error({:unused_args, {:foo,3}, 2})` → "the default value for the last optional argument" (singular) and `{:unused_args,{:foo,3},1}` → "the default values for the last 2 optional arguments". `arity-count` is the number of unused trailing defaults and the plurality matches. Basis: `verified` (ran `elixir -e` against the source-built runtime; output quoted above).

- **`format_error`/`format_diagnostic` coverage of every emitted tuple** — Search space: every `:elixir_errors.*`, `Helpers.warn`, and `undefined_function!` call site in the file (lines 174, 187, 196, 198, 199, 269) emits one of `{:unused_def,fa,kind}`, `{:unused_args,fa[,diff]}`, `{:unused_clause,kind,fa}`, `{:undefined_function|:incorrect_dispatch,fa,mod}`. `kind` at the `unused_def` sites is only ever `:defp`/`:defmacrop` (the `private` list at line 95 collects exactly the non-`:def`/`:defmacro` defs), both of which have clauses. The `{f,a} in [__info__: 1, ...]` guard and `:incorrect_dispatch` clause were exercised above without a `FunctionClauseError`. Basis: `verified`.

- **`min_reachable_default/warn_unused_def` default-arg reachability (lines 177–217)** — `min_reachable_default` scans arities `max..min` and `last` is overwritten on each *used* arity, so it converges to the lowest reachable arity; the `:none/^min/^max/diff` branches map correctly onto whole-function-unused / all-defaults-reachable / all-defaults-unused / partial. `diff` is always strictly between `min` and `max`, so `arity-count ≥ 1` and the format clauses never underflow. Basis: `asserted` (arithmetic traced) plus the `verified` format-message check above.

- **`add_inferred` index accounting (lines 552–559)** — Starting index `total-1` with an `inferred` list whose length is invariantly `total` (grown by `+1` only when `type_index == -1`, lines 486/374) means exhausting the list lands exactly on the `-1` base clause; no `FunctionClauseError`. Insertion-order `type_index` values line up with the final `Enum.reverse(inferred)` (line 492) positions consumed by `skip_unused_clause?`. Basis: `asserted` (traced two- and three-clause cases by hand).

- **`default_local_handler` mapping records the arrow `index` rather than the deduplicated `type_index` (line 375)** — this diverges from the analogous `group_clauses` (lines 486/488) only when two intersection arrows produce identical `clause_types`. Even then no out-of-range `Enum.fetch!` occurs, because `skip_unused_clause?`/`Enum.fetch!(inferred, type_index)` in `warn_unused_clauses` (lines 257, 280) is reached only for indexes present in `pending` (real type indexes ≤ `total-1`), and the super-clause path is additionally excluded from `warn_unused_clauses` by the `not Keyword.get(meta, :from_super, false)` filter (line 249). No consuming path reaches it. Basis: `asserted` (control-flow traced; the consumer is `warnings/6` only, and its `from_super` filter gates this def shape out).

- **Transient `:none` inferred reaching `group_clauses_by_return` (lines 86–89)** — `local_handler/5` returns `{kind, :none, ctx}` only while a `fun_arity` is mid-recursion (present in `local_sigs` as a bare atom, line 293). At the top-level `infer` reduce each def is finalized to a full `{kind, inferred, mapping}` triple (line 305) before the reduce visits it, and `group_clauses_by_return/1` has a total fallthrough clause (line 580) regardless. No crash, no `:none` leaking into `types`. Basis: `asserted` (recursion structure traced).

- **`List.duplicate(dynamic(), arity-1)` underflow in `default_domain` (line 155)** — reachable only inside the `with impl … fun_arity in callbacks` branch, and `callbacks` comes from `protocol.__protocol__(:functions)`, whose entries always have arity ≥ 1 (dispatch arg), so `arity-1 ≥ 0`. Basis: `asserted` (protocol-function arity is a documented Elixir invariant).

- **`and`/`or` vs `&&`/`||` at lines 42, 64** — every operand there is already a boolean (`!= false`, `!= nil`, `==` comparisons, `Keyword.has_key?`, `not protocol?`), so the strict boolean operators are correct and cannot hit the `and`-needs-boolean pitfall. Basis: `asserted`.

## Risks / caveats
- The two real entry points (`infer/7`, `warnings/6`) require live compiler state (`defs`, `cache`, `env`, a populated `stack`) that is impractical to synthesize outside a real compile, so my confidence in the deep inference paths (`group_clauses`, `compute_domain`, `add_inferred`, context threading) rests on control-flow/arithmetic tracing rather than execution; only the `format_*` surface was driven directly.
- The standing folder threat model (Unicode UAX #31 / UTS #39 tokenizer and identifier-security obligations) does **not** apply to this file — `Module.Types` is the gradual-type inference driver, with no source-text, tokenizer, or Unicode handling. Those clauses are out of scope here.

