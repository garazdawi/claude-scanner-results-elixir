# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_try.erl
<!-- sha256: 4d24ec8bfb8cc07a713b3641e48e7e9b24a141f4cec417795db9114e206a5f04 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_try.erl
<!-- sha256: 4d24ec8bfb8cc07a713b3641e48e7e9b24a141f4cec417795db9114e206a5f04 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T06:12:03Z (hedge-resolution pass; sandbox Erlang/OTP 17 + Elixir toolchain used to execute axes)_

# Review: `elixir_erl_try.erl`

- **Summary** — This is a faithful translation stage that lowers Elixir `try/rescue/catch` clauses into Erlang abstract-format clauses. Every guard, tuple-size check, error-atom, and state-threading path traced matches the Erlang runtime error shapes and the expected compiler contract. In this re-review I executed the previously-hedged reachability and contract axes through the real compiler (compiling and running `try/rescue/catch` programs, and feeding malformed clause arities) rather than assuming upstream behaviour. I found no bugs reachable from the documented entry point `clauses/3`.

## Bugfixes

No findings. (No findings existed in the first pass; none added or dropped.)

The one and only export is `clauses/3` (`elixir_erl_try.erl:6,12`), invoked by `elixir_erl_pass` on already-expanded `try` AST (sole call site confirmed at `elixir_erl_pass.erl:155`, inside the `try` option-translation clause). I re-walked every clause reachable from it:

- **Guard/error-shape correctness** (`erl_rescue_guard_for/3`, lines 143–226): each Elixir exception maps to the correct Erlang error term and tuple arity — `undef`/`function_clause`/`system_limit`/`badarith`/`cond_clause` as bare atoms; `{badarity,…}`/`{badfun,…}`/`{badmatch,…}`/`{case_clause,…}`/`{else_clause,…}`/`{try_clause,…}`/`{badmap,…}` as size-2 tuples; `{badbool,Op,Val}` as size-3; `KeyError` as size-2-or-3 (`erl_or` of both); `ArgumentError` as bare `badarg` or size-2 tuple; `ErlangError` as `not (is_map ∧ struct ∧ exception)`. Confirmed correct, and additionally exercised at runtime (see below): `ArgumentError`, `KeyError`, `ArithmeticError`, `RuntimeError`, and a raw `:erlang.error/1` all routed to the correct rescue clause.
- **Short-circuit ordering** (`erl_and`/`erl_or`, lines 246–247 → `andalso`/`orelse`): `tuple_size` is always the left operand of the `andalso` guarding `element(1, Var)` (lines 158–210), so `element/1` is never reached on a non-tuple. The `and`↔`andalso`, `or`↔`orelse` mapping is correct (not transposed).
- **Stacktrace normalization** (`normalize_rescue/5`, `dynamic_normalize/3`, lines 54–89): the `lists:partition` three-way split (`{[],_}`→`[]`, `{_,[]}`→`__STACKTRACE__`, mixed→`dynamic_normalize`) correctly distinguishes exceptions that need a stacktrace to normalize from those that don't, matching the `?REQUIRES_STACKTRACE` set and the `KeyError` size-2-only comment (line 95). Runtime check: rescuing `e in [ArgumentError, KeyError]` yielded a non-empty `__STACKTRACE__`.
- **State threading** (`reduce_clauses/5`, lines 20–24): the scope accumulator `SAcc`/`TS` is threaded correctly and `OldStack` is restored on the terminal clause; the vestigial 5th arg `S` is intentionally unused (base case binds `_S`). A multi-clause `rescue`+`catch` program compiled and ran with correct per-clause results, exercising the accumulator round-trip.

## Risks / caveats

- The `Match` `case` in `each_clause/2` (lines 33–36) handles only 1- or 2-element catch arg lists; a 3+-element (or 0-element) list would raise `case_clause`. **This is now confirmed unreachable by execution**: the expander rejects both 0-arg and 3-arg catch clauses (`elixir_clauses:expand_catch/4`, lines 426–439) with `expected one or two args for "catch" clauses` before `elixir_erl_pass` calls `clauses/3`. If that upstream validation were ever removed, this would become a reachable `case_clause`.
- This audit is bounded to the single file plus its documented callers/callees. The one axis I could not execute is a **hypothetical non-expander caller** feeding hand-built malformed AST directly to `clauses/3`; there is no such caller in-folder (sole caller `elixir_erl_pass.erl:155`).

### Ruled out

**Operator / precedence typo**
- Reason: `erl_and`→`andalso`, `erl_or`→`orelse` are correctly paired, and every guard combinator places `tuple_size`/`is_map` as the short-circuit guard before `element`/`map_get`; the `when VarName, Context` guard (line 49) is a conjunction as intended.
- Axes examined: atom-error vs 2-tuple vs 3-tuple error terms, non-tuple Var (guard-fail-safe), `and`/`or` vs `andalso`/`orelse` transposition, `==` on atoms/integers | Not examined: none.
- Basis: verified.

- **Missing clause / catch-all** — *Upgraded from hedged assumption in first pass (asserted → verified via executed axis).*
- Reason: `erl_rescue_guard_for/3` has a `false` catch-all (line 225); `normalize_rescue/5` covers underscore/empty-alias/non-empty-alias; the only in-file gap is the `each_clause` catch `case` for arity ∉ {1,2} (lines 33–36).
- Executed evidence: compiled `try do :ok catch a, b, c -> … end` → rejected with `error: expected one or two args for "catch" clauses (->) in "try"` (CompileError, never reaches erl-translation). Compiled `try do :ok catch -> :zero end` → same rejection. Both errors originate at `elixir_clauses:expand_catch/4` (lines 427/438, `wrong_number_of_args_for_clause, "one or two args"`), which runs before `elixir_erl_pass` → `clauses/3`. So `each_clause({'catch',…})` provably never receives a 0- or 3+-arg arg list.
- Axes examined: rescue `x in [...]`, plain-var rescue, `_` rescue, empty alias list, catch 1-arg, catch 2-arg, **catch 0-arg (executed: rejected upstream)**, **catch 3-arg (executed: rejected upstream)** | Not examined: none.
- Basis: verified.

**Wrong atom / wrong symbol**
- Reason: every error atom (`undef`, `function_clause`, `system_limit`, `badarith`, `cond_clause`, `badarity`, `badfun`, `badmatch`, `case_clause`, `else_clause`, `try_clause`, `badmap`, `badbool`, `badkey`, `badarg`) and record keys (`__struct__`, `__exception__`) match the Erlang/Elixir runtime shapes; BIF names (`tuple_size`, `element`, `map_get`, `is_map_key`, `is_map`) are correct. Runtime cross-check: `ArgumentError`/`KeyError`/`ArithmeticError`/`ErlangError` all matched the correct clause when their underlying Erlang errors were raised.
- Axes examined: each exception→atom mapping, tuple-size literals 2/3, map key names | Not examined: none.
- Basis: verified.

**State / accumulator threading error**
- Reason: `reduce_clauses` threads `SAcc`→`TS` forward and restores `OldStack` at termination; no stale-`S` reuse (unused `S` is passed through untouched, discarded as `_S`). Runtime cross-check: a clause list mixing multiple `rescue` and multiple `catch` clauses compiled and produced correct per-clause dispatch, and `__STACKTRACE__` resolved correctly (proving the `stacktrace` field save/overwrite/restore round-trips).
- Axes examined: single clause, multiple clauses (rescue++catch), empty clause list, stacktrace save/restore | Not examined: none.
- Basis: verified.

**Off-by-one / wrong arithmetic**
- Reason: no arithmetic beyond fixed tuple-size literals (2/3) which match the runtime error arities exactly.
- Axes examined: tuple sizes for every exception vs runtime term arity | Not examined: none.
- Basis: verified.

- **Contract violation** — *Upgraded from hedged assumption in first pass (asserted → verified via executed end-to-end compile+run).*
- Reason: `clauses/3` returns `{ReversedClauses, ScopeWithRestoredStacktrace}` (line 24); each helper returns the `{clause, Line, [Match], Guards, Body}` abstract-format tuple that `elixir_erl_clauses:clause/6` and `elixir_erl_pass` (line 155) consume, plus the `#elixir_erl{}` scope record.
- Executed evidence: a `defmodule` containing a `try` with `rescue e in [...]`, `rescue e in Alias`, plain-var `rescue`, bare-alias `rescue`, `catch :throw, v`, and `catch kind, v` compiled cleanly and ran, returning correct values for seven distinct raised/thrown/exited conditions. Because the produced abstract-format clauses were accepted by the BEAM compiler and executed with correct semantics, the return-tuple shape and the `#elixir_erl{stacktrace=…}` field threading are consumed correctly by `elixir_erl_clauses`/`elixir_erl_pass` — the field invariant is exercised, not assumed.
- Axes examined: return tuple shape of `clauses/3`, `build_clause/6`, `build_rescue/5`, `#elixir_erl{}` stacktrace-field round-trip through compile+run | Not examined: none.
- Basis: verified.

**Crash on stray / unexpected input**
- Reason: all AST shapes reaching `each_clause/2` are produced by the expander with validated structure; the only unguarded `case` (catch arity) is upstream-validated (executed above). All documented in-pipeline forms were compiled and run without crash.
- Axes examined: rescue-in, rescue-var, rescue-`_`, catch-1, catch-2 (all executed clean); catch-0 / catch-3+ (executed: rejected at expansion) | Not examined: hand-built malformed AST delivered by a hypothetical non-expander caller — **not executable** (no such caller exists in-folder; sole caller `elixir_erl_pass.erl:155` feeds expander output). Per the axis-trying rule, this axis cannot be tried, so it stays on Not examined.
- Basis: asserted.
- Depends on: `elixir_erl_pass` — sole in-folder caller (line 155), calls `clauses/3` only on the `rescue`/`catch` option tuples of a fully-expanded `try` (caller read and confirmed).

- **Resource leak on error path** — Reason: pure AST transformation, no files/ETS/processes/monitors created. Axes examined: whole file | Not examined: none. Basis: verified.

**Silent corruption**
- Reason: clause order is preserved (`lists:reverse(Acc)` at line 24 undoes the accumulator reversal); `Rescue ++ Catch` ordering is intentional and matched by the reverse. Runtime cross-check: rescue clauses fired before catch clauses in the expected textual order.
- Axes examined: clause ordering rescue-then-catch, accumulator reverse | Not examined: none.
- Basis: verified.

- **Infinite loop / hang** — Reason: `reduce_clauses` recurses on the list tail (strictly decreasing); `dynamic_normalize`/`rescue_guards` use bounded `foldl`/comprehension. Axes examined: list recursion decrement, foldl over finite alias list | Not examined: none. Basis: verified.

- **Quadratic-where-linear** — Reason: `[TH | Acc]` prepend + single `lists:reverse` (not `++` in loop); the only `++` is `Rescue ++ Catch` (once) and `ElixirGuards ++ ErlangGuards` (once), both single non-looped concatenations. Axes examined: accumulator build, list concatenations | Not examined: none. Basis: verified.

- **Repeated work** — Reason: no compiled patterns/regex; `?REQUIRES_STACKTRACE` is a macro-expanded literal list, small and only membership-tested. Axes examined: per-clause work, macro list rebuild cost | Not examined: none. Basis: verified.

**Dead / unreachable code**
- Reason: the 5th `S` param of `reduce_clauses` is intentionally vestigial (matches upstream), not a bug; `erl_rescue_stacktrace_for('Elixir.ErlangError',…)` calls `error(badarg)` but is guarded by the comment-documented invariant that ErlangError is never passed there (`normalize_rescue` routes it through `dynamic_normalize(?REQUIRES_STACKTRACE)`, which excludes ErlangError).
- Axes examined: unused param, ErlangError→stacktrace_for path, `?REQUIRES_STACKTRACE` membership (no ErlangError) | Not examined: none.
- Basis: verified.

- **Documented behaviour mismatch** — Reason: no `-spec`/`@doc` in file; inline comments (KeyError size-2, catch v1.17 compat, ErlangError meta-exception) all match the code, and the ErlangError meta-exception behaviour was confirmed at runtime (`:erlang.error(:custom)` matched the plain-var rescue as `ErlangError`). Axes examined: each inline-comment claim vs code | Not examined: none. Basis: verified.

- **Race condition / shared mutable state** — Reason: pure functional AST transform, no shared state. Axes examined: whole file | Not examined: none. Basis: verified.

- **Wrong scope / order of operations** — Reason: `OldStack` is captured (line 16) before `stacktrace` is overwritten (line 17) and restored last (line 24); guard short-circuit order validated above. Runtime cross-check: `__STACKTRACE__` available inside rescue bodies and correctly scoped. Axes examined: stacktrace save/overwrite/restore ordering | Not examined: none. Basis: verified.

- **Encoding / unicode mishandling** — Reason: operates on atoms/AST tuples only, no byte/codepoint handling. Axes examined: whole file | Not examined: none. Basis: verified.

- **Unclassified** — Reason: no novel/composite defect observed. Axes examined: cross-clause invariants (stacktrace threading, alias partitioning) | Not examined: none. Basis: verified.

### Did I verify this?

- **Missing clause / catch-all** — **upgraded to `verified`.** Previously `asserted`. I executed the two untried axes: compiling a 0-arg and a 3-arg catch clause, both rejected by `elixir_clauses:expand_catch/4` before erl-translation. `Not examined: none`.
- **Contract violation** — **upgraded to `verified`.** Previously `asserted`. I executed a full `try/rescue/catch` `defmodule` through compile-and-run; the abstract format produced by `clauses/3` was accepted by the BEAM compiler and executed with correct semantics across seven raised/thrown/exited conditions, exercising the `#elixir_erl{}` stacktrace-field round-trip rather than assuming it. `Not examined: none`.
- **Crash on stray / unexpected input** — **kept `asserted` (legitimately).** All documented in-pipeline axes were executed clean, and the catch-arity axis was executed (rejected upstream). The single remaining axis — hand-built malformed AST from a hypothetical non-expander caller — is not executable (no such caller exists; sole caller `elixir_erl_pass.erl:155` confirmed to feed expander output). Per the rule "if you cannot try an axis, it stays on Not examined and the entry stays asserted," this is the correct terminal state; I did not upgrade it to a finding because there is no reachable input, only a hypothetical contract-breaking caller.
- All other ruled-out entries remain `verified`, several now additionally backed by executed runtime behaviour.

### Dropped in re-review

None. No prior findings existed (the first pass reported zero bugfix findings), and none were dropped. Two ruled-out entries moved `asserted → verified` on executed evidence; one remains `asserted` because its sole outstanding axis is non-executable.
