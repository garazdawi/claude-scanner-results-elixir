# /Users/lukas/code/claude-scanner/elixir/lib/eex/lib/eex/smart_engine.ex
<!-- sha256: 773cb44eb882ab501b13cafe9832049f803905327e8c318efa6555cc832e46ef -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:04:41Z_

## Summary
`EEx.SmartEngine` is a thin set of `defdelegate`s to `EEx.Engine` plus a `handle_expr/3` that prewalks `@assign` rewrites; the documented doctest passes and no reachable defect was found in this file.

## Bugfixes

No findings.

### Ruled out

- **`handle_expr/3` prewalk dropping/mis-threading state** — `handle_expr/3` (lines 58–61) only transforms the `expr` AST via `Macro.prewalk(expr, &EEx.Engine.handle_assign/1)` and forwards the *unchanged* `state` and `marker` to `EEx.Engine.handle_expr/3`. State is never dropped. Basis: verified — `EEx.eval_string("<%= @foo %>", assigns: [foo: 1])` returns `"1"` (the moduledoc doctest at line 14), exercising this exact path.
- **Moduledoc doctest mismatch (line 14)** — the only executable doctest in the file. Basis: verified — ran `EEx.eval_string("<%= @foo %>", assigns: [foo: 1])`, output `"1"`, matching the documented result.
- **`@impl true` on a non-callback / wrong-arity delegate** — every `defdelegate`/`def` (`init/1`, `handle_body/1`, `handle_begin/1`, `handle_end/1`, `handle_text/3`, `handle_expr/3`) matches an `EEx.Engine` behaviour callback. Basis: verified — the module compiles without `@impl`/behaviour warnings and the doctest, which routes through `init`→`handle_expr`→`handle_body`, succeeds.
- **Missing-assign behavior** — the actual `@foo` → `fetch_assign!` rewrite and its missing-key handling live in `EEx.Engine.handle_assign/1`, not in this file; out of scope for a per-file finding here.

## Risks / caveats
- This file contains almost no independent logic: five `defdelegate`s and one wrapper. The substantive behavior (assign resolution, missing-assign warning-vs-raise, state shape validation) resides in `EEx.Engine`, which must be reviewed separately to cover the `SMART_ENGINE_ASSIGNS_CONTRACT` sub-claim — it cannot be adjudicated from this file alone.
- The second moduledoc example (`function_from_file` / `Sample.sample(a: 1, b: 2)`, lines 25–36) is illustrative prose, not a runnable doctest, so it is not executed by the test suite; its correctness was not machine-verified here.

