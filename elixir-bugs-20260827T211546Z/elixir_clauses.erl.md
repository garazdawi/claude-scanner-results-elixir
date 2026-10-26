# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_clauses.erl
<!-- sha256: 9163555fb9de0cd1bece71062fa361734121dfc98a6ff434ee26361338c65436 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_clauses.erl
<!-- sha256: 9163555fb9de0cd1bece71062fa361734121dfc98a6ff434ee26361338c65436 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (neighbours read: `elixir.hrl`, `elixir_env.erl`, `elixir_expand.erl`, `elixir_def.erl`, `elixir_fn.erl`, `elixir_utils.erl`)._

# Bug-hunting review — `elixir/lib/elixir/src/elixir_clauses.erl`

## Summary
After a full read of this clause-handling module **and** the callgraph neighbours listed in the standing context, I found no defensible bug reachable through its documented (compiler-internal) public API. The state-threading, guard heuristics, and error paths are faithful to their contracts, and every cross-file contract the first pass had only *assumed* is now confirmed by reading the neighbour that supplies it.

## Bugfixes

No findings.

The exported surface (`parallel_match/4`, `match/6`, `clause/6`, `def/3`, `head/4`, `'case'/4`, `'receive'/4`, `'try'/4`, `'cond'/4`, `with/4`, `format_error/1`) is invoked only by `elixir_def`, `elixir_expand`, `elixir_fn` (confirmed by grepping every `elixir_clauses:` call site under `src/`) with already-parsed Elixir AST plus the internal `#elixir_ex{}` scope record and env map. Notable near-misses that remain NOT bugs:

- `warn_zero_length_guard/2` (239–251): `when Op == '=='; Op == '>'` targets exactly `length(x) == 0` / `length(x) > 0` and maps each to the matching message; a missed case only suppresses a *warning*.
- `assert_at_most_once/4` (535–541): starts at `Count=0`, raises only on the *second* matching key (`Count=1`), so "at most once" is exact.
- `format_error({recursive, Vars, _})` (565–586): no `[]` clause, but every call site passes a non-empty `Vars` (`[Current]`, `[Current,Key]`, `[Current|keys]`), so the missing empty clause is unreachable.
- `normalize_rescue([])` (491–494) returns `[]` (not `false`), so `rescue x in []` is a match-nothing clause, not a crash.
- `def/3` (190–197) returns a **4-tuple** `{Meta, EArgs, EGuards, EBody}`, not the 3-tuple the other exports return — this is correct: its sole caller `elixir_def:store_definition/10` (elixir_def.erl:198) collects those 4-tuples into `Clauses` and feeds them to `store_definition/9`. Not a contract violation.

## Risks / caveats
- One residual: the parallel-match cycle detector (`store_cycles/3`, `validate_cycles/2`, `recur_cycles/8`) is intricate graph logic. I verified its termination (recursion is guarded by `is_map_key(Current, Seen)` and traverses a finite `Cycles` map built by `store_cycles` over a finite write list) and that its structure matches its own comments. I did **not** execute a property test over deeply nested `{a=b, b={:ok,a}} = expr` shapes; that execution axis is unrun and the corresponding entries stay `asserted`. No structural defect was found.

### Ruled out

**Operator / precedence typo**
Reason: Guards use `;`-alternation only in `warn_zero_length_guard` (`Op == '=='; Op == '>'` and `Op == 'or'; Op == 'and'`), each a simple OR of equality tests with no `,`/`;` mixing.
Axes examined: `==` vs `>` operand, `and`/`or` recursion, literal-`0` second arg | Not examined: none.
Basis: verified.

**Missing clause / catch-all**
Reason: Every public dispatcher (`'case'`, `'cond'`, `'receive'`, `'try'`, `expand_*`) has explicit empty/`not is_list`/unexpected-option/`bad_or_missing_clauses` fall-throughs raising a formatted `file_error`, not a `function_clause`.
Axes examined: empty opts `[]`, non-list opts, unknown option key, empty clause list `{Key,[]}`, multiple `after` | Not examined: none.
Basis: verified.

- **Wrong atom / wrong symbol** — *Upgraded from hedged assumption in first pass (now verified by reading `elixir.hrl`).*
Reason: Option atoms (`do`, `'after'`, `'catch'`, `'rescue'`, `'else'`), error tags, and record-field names all match their raise/`format_error` sites. The one field-spelling axis the first pass left open is now confirmed: `elixir.hrl:13–52` defines `#elixir_ex{}` with fields `prematch`, `stacktrace`, `unused`, `vars`, `version` spelled exactly as destructured here (lines 17, 33, 158–166, 421).
Axes examined: each `format_error` tag vs its raise site; every `#elixir_ex{}` field name used here vs the record definition in `elixir.hrl` | Not examined: none.
Basis: verified. (Cite: `elixir.hrl:13–52`.)

- **State / accumulator threading error** — *Upgraded from hedged assumption in first pass (field-preservation now verified by reading `elixir_env`).*
Reason: `with/4`, `def/3`, `match/6`, `parallel_match/4`, `expand_clauses_origin/6` thread `S0/S1/SA/SG/SB` forward without reusing a stale binding (verified in-file). The cross-file half — that the `elixir_env` helpers return a `#elixir_ex{}` preserving unrelated fields — is confirmed: `reset_read/2` (env:69), `reset_unused_vars/1` (env:97), `check_unused_vars/2` (env:100), `merge_and_check_unused_vars/3` (env:114) all use `S#elixir_ex{...}` record-update syntax, which preserves every field not named; `merge_vars/2` (env:85–93) returns a plain map, exactly as `parallel_match/4` line 35 consumes it.
Axes examined: `with` do/else env threading; `def` prematch reset between args/guards/body; mapfoldl accumulator in `expand_clauses`; parallel_match write accumulation; neighbour helpers preserve unrelated `#elixir_ex{}` fields | Not examined: none.
Basis: verified. (Cite: `elixir_env.erl:69,85–93,97,100,114–117`.)

**Off-by-one / wrong arithmetic**
Reason: Only arithmetic is `Count + 1` in `assert_at_most_once` (triggers on the 2nd key) and `Counter+1` for the `with`/`fn` version bump (fresh counter after use); no index/length math.
Axes examined: zero/one/two duplicate keys, version counter reuse | Not examined: none.
Basis: verified.

- **Contract violation** — *Upgraded from hedged assumption in first pass (caller shapes now read).*
Reason: Each export returns the shape its caller destructures. Confirmed by reading callers: `clause/6` returns `{{'->',…}, SR, ER}` and is destructured as `{EClause, SAcc, EAcc}` at `elixir_fn.erl:20–23` and `elixir_expand.erl:912–915`; `head/4` / `match/6` return 3-tuples destructured at `elixir_expand.erl:1102` and `:1109–1112`; `def/3` returns the 4-tuple `{Meta, EArgs, EGuards, EBody}` collected by `elixir_def.erl:198`; `with/4`, `'case'/4`, etc. return `{Expr, Scope, Env}` to `elixir_expand`. Error paths raise via `file_error`.
Axes examined: return arity/shape of `match/6`, `head/4`, `clause/6`, `def/3`, `with/4` vs each caller's destructuring pattern | Not examined: none.
Basis: verified. (Cite: `elixir_fn.erl:20–23`, `elixir_def.erl:198`, `elixir_expand.erl:912–915,1102,1109–1112`.)

- **Crash on stray / unexpected input** — *Upgraded from hedged assumption in first pass (caller contexts now enumerated).*
Reason: `match/6` only matches `#{context := nil}`. I enumerated every caller: the only external callers are `elixir_expand.erl:19` and `:1109`, plus in-file `head/4`/`expand_rescue`/`guarded_head`. At `elixir_expand.erl:16–19` the `=`-expansion runs `assert_no_guard_scope` (line 17) *before* calling `match/6`, so `guard` context is rejected and only `nil` reaches it; `:1109` is a comprehension-generator body (nil context); the in-file paths (`head/4`←`clause/6`/`fn`, `expand_with`, `expand_catch`, `expand_rescue`) all run under try/with/case/for bodies whose env is nil. No caller in the callgraph passes `context =/= nil`. Malformed clause shapes fall through to `clause/6`/`expand_catch`/`expand_rescue` error clauses.
Axes examined: non-`->` clause body; wrong-arity `when` in catch; non-list receive after; every enumerated `match/6` caller's context value | Not examined: none.
Basis: verified. (Cite: `elixir_expand.erl:13–19,1099–1112`; grep of all `elixir_clauses:match`/`:head` call sites.)

**Resource leak on error path**
Reason: Allocates no files, ETS, processes, monitors, or counters; pure AST/scope transformation, `file_error` throws.
Axes examined: whole-file scan for ETS/port/spawn/monitor/open | Not examined: none.
Basis: verified.

**Silent corruption (accumulators)**
Reason: `EArgs ++ [EGuard]`, `[EGuard | EGuards]`, `maps:merge(Acc, Write)` preserve all elements; no truncation or empty-collection short-circuit drops data.
Axes examined: empty args list in `guarded_head`, single vs multi clause fold, empty write map in `store_cycles` | Not examined: none.
Basis: verified.

**Silent corruption (cycle graph)**
Reason: `store_cycles/3` accumulates `Write`/`DependsOn` via `maps:merge`/`maps:fold` with an `error`-tagging `merge_with`; structurally consistent with its documented algorithm (comments 47–91). No neighbour is involved — this is in-file graph logic.
Axes examined: write-set accumulation, skip-list growth, `error`-tag propagation vs comments | Not examined: execution/property test over deeply nested parallel-match cycle shapes (not run — no Erlang harness stood up for the isolated graph functions).
Basis: asserted.

**Infinite loop / hang**
Reason: All recursion (`unpack_match`, `store_cycles`, `guards`, `recur_cycles`, `assert_at_most_once`, `warn_catch_before_rescue`) decreases on list tails or on finite `Seen`/`Cycles` key sets; `recur_cycles` guards re-entry with `is_map_key(Current, Seen)`, and `Cycles` is finite because `store_cycles` folds a finite write list.
Axes examined: `recur_cycles` revisit guard, list-tail recursions, finiteness of `Cycles`/`Seen` | Not examined: execution over a pathological cyclic graph where `Seen` growth vs skip-list interaction is adversarial (not run).
Basis: asserted.

**Quadratic-where-linear**
Reason: `EArgs ++ [EGuard]` and `maps:keys/from_keys` operate on small per-clause collections; no `Acc ++ X` in an unbounded loop, no per-iteration `length/1`; `lists:member(Key, OnceKeys)` is over a tiny per-node set.
Axes examined: append sites, member-as-set usage, fold bodies | Not examined: none.
Basis: verified.

**Repeated work**
Reason: No compiled patterns (`binary:compile_pattern`, `re:compile`) or repeated file/BEAM scans; `origin/2` does a single `lists:keyfind` per call.
Axes examined: whole-file scan for compile/re/ets:fun2ms | Not examined: none.
Basis: verified.

**Dead / unreachable code**
Reason: All clauses reachable via some AST shape; the only structurally-unreachable branch (`format_error` empty-`Vars`) is intentional defensive coverage.
Axes examined: `format_error` arms, `expand_catch` arity arms, receive after arms | Not examined: none.
Basis: verified.

**Documented behaviour mismatch**
Reason: Only a header comment ("Handle code related to args, guard and `->` matching…"); no `-spec`/`@doc` prose to contradict, and the code matches it.
Axes examined: module comment vs export set | Not examined: none.
Basis: verified.

**Race condition / shared mutable state**
Reason: Pure functional transformation on caller-owned scope records; no global/ETS/process-registry mutation.
Axes examined: whole-file scan for system_flag/ets/register | Not examined: none.
Basis: verified.

**Wrong scope / order of operations**
Reason: `def/3` validates cycles before guard/body expansion; `with/4` expands clauses → do → else in dependency order; `file_error`/`file_warn` fire only after their inputs are computed.
Axes examined: `def` cycle-check ordering, `with` do/else ordering, try warn-before-expand | Not examined: none.
Basis: verified.

**Encoding / unicode mishandling**
Reason: No byte/codepoint handling here; identifier/string bytes are handled upstream in the tokenizer/interpolation.
Axes examined: whole-file scan for binary/charlist ops | Not examined: none.
Basis: verified.

**Unclassified**
Reason: No novel composite defect surfaced. The cycle detector is the only candidate and is self-consistent with its documented algorithm. `parallel_match/4` is entered only from `elixir_expand.erl:14` (context `match`), and `elixir_expand:expand/3` populates `prematch` write-sets recursively before `store_cycles` reads them; the write-set *shape* (`{Read, {Cycles, SkipList}, Meta}`) is confirmed against `elixir.hrl:22` and the `def/3` seed at line 191.
Axes examined: parallel-match cycle/skip-list logic vs its comments; prematch tuple shape vs header | Not examined: execution/fuzzing of nested parallel-match cycles (not run).
Basis: asserted.

### Did I verify this?

- **Wrong atom/symbol** — upgraded `asserted → verified`. Read `elixir.hrl:13–52`; every `#elixir_ex{}` field this file destructures is present and spelled identically. `Not examined: none`.
- **State/accumulator threading**, **Contract violation**, **Crash on stray input** — upgraded `asserted → verified`. These were the first pass's `Depends on:` hedges. Each is a cross-file *shape / caller-enumeration* fact, which is exactly what option (a) resolves by reading: `elixir_env.erl` helpers are record-updates preserving fields (`env:69–117`); every export's return shape matches its caller's destructuring (`elixir_def.erl:198`, `elixir_fn.erl:20–23`, `elixir_expand.erl:912–915,1102,1109–1112`); `match/6`'s only callers all pass `context := nil` (guarded at `elixir_expand.erl:17`). No differential-input axis remained on these, so `Not examined: none` is honest.
- **Silent corruption (cycle graph)**, **Infinite loop**, **Unclassified** — kept `asserted`. Their residual axis is *execution* over adversarial nested parallel-match cycle shapes. Reading the neighbours does not touch this — the untried axis lives in running the graph functions, which I did not do (no isolated Erlang harness for `store_cycles`/`recur_cycles`, which call `elixir_errors:file_error/4`). Per the re-review rule, reading cannot clear an execution axis, so these stay `asserted` with the untried axis named. No bug evidence surfaced, so none is upgraded to a finding.
- **All other entries** — kept `verified`; every axis on their in-file datatypes was traced.

### Dropped in re-review

None. No prior finding existed (the first pass reported none), and no prior dismissal was weakened. Three dismissals moved `asserted → verified` on the strength of the neighbour reads cited above; the three execution-bound cycle-detector dismissals were retained as `asserted` rather than being over-claimed as verified.
