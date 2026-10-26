# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/expr.ex
<!-- sha256: e2f9b066f71a5242bf797cdd1691cfe856d2f4ba5bbcec867ddb6945c3055942 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:27:25Z_

I've confirmed the file is byte-for-byte identical to the reference build at `/opt/elixir` (and `/cscan/0`), so the installed compiler exercises exactly this source. I ran the type checker (which invokes `Module.Types.Expr.of_expr/5` on every compiled expression) across the full range of `of_expr` forms — including the diagnostic-emitting paths — with no crashes and correct output.

# Summary
After static review and runtime exercise of every `of_expr/5` clause and the `format_diagnostic/1` paths against a compiler built from this exact source, I found no genuine, reachable bug in `expr.ex`.

# Bugfixes

No findings

### Ruled out

- **`list_tl` case at lines 100–104 is non-total (would `CaseClauseError`)** — The `hd_type` case (89–92) has a `_ ->` catch-all but the `tl_type` case only handles `{:ok, type}` and `:badnonemptylist`. Verified in `descr.ex:2493-2515`: `list_tl/1` returns *only* `{:ok, _}` or `:badnonemptylist` for any input, so the two clauses are exhaustive. Basis: verified (read callee).
- **`apply_many([], fun, ...)` (line 913) calls `Apply.remote` with 7 args while other clauses use 8** — Looked like a wrong-arity/argument-order bug (passing `fun` where a module is expected). Verified in `apply.ex:372` and `apply.ex:386`: both `remote/7` (documented "entry point for applying functions without a known module") and `remote/8` are defined; the 7-arg call is the intended no-known-module path. Basis: verified (read callee).
- **`format_diagnostic/1` tuple shapes mismatching their throw/`error` sites** (`:badmap`/`:badkey`/`:baddomain`/`:badupdate`/`:badtimeout`/`:badbitgenerator`/`:badbitbody`/`:badcond`) — Cross-checked each throw (lines 197–199, 230, 598, 826, 633) against its formatter (1065–1249). Drove `:badmap`, `:badupdate`, `:badtimeout`, `:badbitgenerator` end-to-end via `t2.ex`; all produced correct, well-formed diagnostics with no arity error. Basis: verified (ran, observed output).
- **Compiler crash on valid but less-traveled forms** (map update, struct update, `with/else`, `receive/after`, `for ... into:` for bitstring/list/map, `try` with `rescue`/`catch`/`else`/`after`, `__STACKTRACE__`, captures `&Enum.map/2`/`&(&1+1)`, chained match `a = b = x`, improper list `[1,2|t]`, `m.key`) — Compiled all via `t1.ex`/`t4.ex`; every one type-checked cleanly with no exception raised inside `Module.Types.Expr`. Basis: verified (ran).
- **Reverse-arrow / cache machinery (lines 384–508, 958–985) mis-threading state or missing a cache key** — Exercised nested `case`, `case` as a match RHS, `case` inside `with` clause heads, and multi-clause `cond` via `t3.ex`; all compiled without `KeyError`/`CaseClauseError` from `Map.fetch!(context.reverse_arrows, version)` or `cache_result`/`cache_arrows`. Basis: verified (ran) for absence of crash; the finer var-scoping *semantics* were reviewed but not independently oracle-checked.
- **Two-clause `cond` context selection (lines 336–340) restoring the wrong var set** — Reviewed each branch (`empty?(pos_body_type)` → falsy path, `empty?(neg_body_type)` → truthy path, else → original `acc_context`); each restores vars consistent with which branch is statically reachable. Basis: asserted (reasoned; no behavioral divergence observed in `t3.ex`).
- **Documentation/`@spec` contract mismatch** — The file declares no `@spec`, `@doc`, or doctest (only `@moduledoc false`), so there is no executable documentation contract to violate. Basis: verified (full-file read).

# Risks / caveats
- This is unmodified current Elixir source; the runtime checks confirm robustness on the inputs I drove, but the type-inference *results* (as opposed to crash-freedom) are only spot-checked — a subtle imprecision that produces a wrong-but-non-crashing inferred type would not surface in these tests and is outside what I could adjudicate without a type oracle.
- The two-clause-`cond` and reverse-arrow variable-scoping branches are the highest-complexity logic here; I verified they don't crash and reviewed their intent, but a semantic soundness bug in var threading there would require dedicated equivalence testing against the general path to rule out fully.

