# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_clauses.erl
<!-- sha256: 9163555fb9de0cd1bece71062fa361734121dfc98a6ff434ee26361338c65436 -->

_Analysed on 2026-08-27T14:40:52Z_

I have enough to conclude. All defensive clauses yield clean `CompileError`s (reachable, correct, not dead), the file contains no deserialisation/file/path/command/network/crypto/NIF/atom/ETS/process-dict sinks, and its only "code execution" is the compiler's own macro expansion — reachable exclusively through compile/eval entry points that are documented to run arbitrary code, never through the parse-only `Code.string_to_quoted`.

# Security review — `elixir_clauses.erl`

## Summary
`elixir_clauses.erl` is a compile-time AST-expansion helper for `case`/`cond`/`receive`/`try`/`with`/`fn`/`def` clause handling and parallel-match cycle detection; it holds no deserialisation, file, network, crypto, atom-creation, or shared-state sinks, and every dangerous-looking path is reachable only through the documented compile/eval APIs that already run arbitrary code — so there are no security findings.

## Security
No findings.

This module is invoked only from `elixir_expand`, `elixir_def`, and `elixir_fn` (verified via callgraph and `grep elixir_clauses:`), i.e. exclusively during **macro expansion / compilation**. The parse-only public entry point `Code.string_to_quoted` (→ `elixir:string_to_quoted` → tokenizer + parser) never reaches this file. Every path that could be abused (macro expansion, unbounded cycle-graph work, non-terminating rescue expansion) sits behind `Code.compile_string` / `Code.eval_string` / `Code.eval_quoted` / `Macro.expand*`, all documented to execute arbitrary code at compile time. An attacker who can drive these sinks already has compile-time code execution, so none of them grants new capability. Combined with the total absence of the classic dangerous sinks (confirmed by grep — no `binary_to_term`, `*_to_atom`, `file:`/`filelib:`, `os:cmd`/`open_port`, `ets:`, process dictionary, `crypto:`, or NIFs), the file yields nothing meeting the bar.

## Risks / caveats
- **The whole file lives behind the "compilation runs code" boundary.** The review's central load-bearing claim is that `elixir_clauses` is unreachable from parse-only APIs and only reachable from compile/eval APIs. This was verified against the in-folder callgraph and `elixir_expand.erl` call sites; if a *future* caller wired any of these functions into a parse-only or otherwise "sandboxed" path, the compile-time DoS vectors below would become in-scope.
- **Parallel-match cycle detection is polynomial but not a distinctive DoS.** `store_cycles`/`validate_cycles`/`recur_cycles` were stress-tested; on pathological inputs they stay within a constant factor (~2.7×) of ordinary large-pattern compilation, which is itself super-linear for unrelated back-end reasons. Not attributable to this file.
- **Compiler diagnostics echo attacker-influenced source snippets** (`'Elixir.Macro':to_string/1` into `~ts` in `format_error/1`). This is by-design and identical to every compiler; a build system that renders untrusted-compile diagnostics into a terminal/log inherits the generic diagnostic-echo concern, which is out of this file's control.

### Ruled out

- **Macro expansion / code execution** — `'Elixir.Macro':expand_once/2` (line 482), `elixir_expand:expand/3` and `expand_args/3` (throughout).
Reason: these are the compiler expanding the source it was asked to compile; reachable only via `Code.compile_string`/`eval_string`/`eval_quoted`/`Macro.expand*`, all documented to run arbitrary code, so exploiting them yields no capability beyond what the entry point already grants.
Axes examined: reachability paths — parse-only (`Code.string_to_quoted` → does not reach this module), compile/eval (reaches, documented code-exec) | Not examined: none.
Basis: verified (traced callgraph + `elixir_expand.erl:13-14,819,914,1102,1109`; `grep elixir_clauses:` shows only expansion-phase callers).

- **Parallel-match cycle detection + `unpack_match` recursion** — `parallel_match/4`, `store_cycles/3`, `validate_cycles/4`, `recur_cycles/8` (lines 16-153). Class: Resource consumption.
Reason: dispositively behind the compile-time/code-exec boundary (no uplift); additionally polynomial — the `is_map_key(Current, Seen)` gate (line 99) caps total node processings at V, and measured scaling stays within a constant factor of a no-`=` control pattern of equal size.
Axes examined: single large cycle, long alias chain, independent pairs, repeated 1-degree cycle, plain-tuple control, sizes to 8000 vars | Not examined: none that change the ruling (the boundary alone is dispositive).
Basis: verified (ran scaling + control-isolation; read the memoization gate).

- **`expand_rescue` re-expansion loop** — `expand_rescue({_, Meta, _} = Arg, …)` → `Macro.expand_once` → recurse (lines 481-485). Class: Resource consumption / non-termination.
Reason: the `Arg -> false` fixpoint check terminates ordinary cases; only a user-authored macro that emits a different term on every expansion loops, which is a self-inflicted compile-time hang behind the code-exec boundary.
Axes examined: fixpoint-reaching macros (terminate), non-3-tuple args (wrapped at line 488, terminate) | Not examined: an actual divergent/alternating macro repro (would hang; not run).
Basis: asserted. Depends on: `Elixir.Macro.expand_once/2` — performs a single expansion step and returns the input unchanged at a fixpoint; and compile-time-only reachability via `elixir_expand`.

- **`format_error/1` diagnostics** — `io_lib:format` with `'Elixir.Macro':to_string(Expr)` values (lines 558-630). Class: Template/interpolation, Human-in-the-loop.
Reason: all format strings are compile-time string literals; attacker-influenced values reach only `~ts`/`~p` argument positions (never the format-string position), so there is no format-directive injection, and source-snippet echo in compiler errors is by design.
Axes examined: `~`-directive injection via data (data cannot reach the format string), newline/escape content in echoed snippets (by-design source echo) | Not examined: downstream terminal/log rendering of diagnostics (out of file).
Basis: verified (read every `format_error` clause; all first args are literals).

- **Validation predicates** — `normalize_rescue/1` (491-494), `assert_at_most_once/4` (535-541), `validate_cycles/4` (93-96). Class: Validation.
Reason: they enforce compile-time well-formedness of the developer's own clause syntax, not a safe/unsafe decision on external data crossing a trust boundary; a wrong answer produces a compile error or rejects valid code (correctness), never a security bypass.
Axes examined: list-of-atoms vs non-atom-list (`normalize_rescue`), duplicate-option counting/order (`assert_at_most_once`) | Not examined: none security-relevant.
Basis: verified (read).

- **`version` counter increment** — `S3#elixir_ex{version=Counter+1}` (line 337). Class: Counter rollover / wrap.
Reason: Erlang integers are arbitrary-precision bignums; `Counter+1` cannot wrap or overflow.
Axes examined: width, overflow | Not examined: none.
Basis: verified (Erlang integer semantics).

- **Defensive/arity-guard clauses** — `expand_catch` ≥4-arg `when` (426), `expand_head` ≥3-arg `when` (500), `multiple_after_clauses_in_receive` (314), `invalid_rescue_clause` (446), `with` unexpected option (329), `case`/`cond`/`receive`/`try` missing/invalid options (255-402). Class: Dead / unreachable defensive code.
Reason: each is reachable from a documented construct and produces a clean `CompileError` (not an uncaught crash), so none is dead and none is a crash vector.
Axes examined: over-arity `when`-clauses, duplicate `after`, non-atom rescue lists, unknown `with` option, empty `case` | Not examined: none.
Basis: verified (ran `defs.exs`; all six inputs → `{:clean_error, CompileError}`).

- **Absent sink classes** — Deserialisation, File ops, Path handling, Archive extraction, Command execution, Network, Cryptography, Memory safety/NIFs, Shared mutable state (process dict / ETS / app env), Concurrency, Atom-table exhaustion (`*_to_atom`).
Reason: none of these call shapes exist in the file.
Axes examined: full-file grep for `binary_to_term|*_to_atom|open_port|os:cmd|file:|filelib:|xmerl|crypto:|rand_uniform|spawn|ets:|put(/get(/erase(|:consult|load_nif` | Not examined: none.
Basis: verified (ran grep; only `maps:get`/`maps:merge`/`map_get` matched).

### Did I verify this?

- **Macro expansion / code execution** — verified; kept. Reachability was traced through the in-folder callgraph and `elixir_expand.erl` call sites, and the "parse-only path does not reach here" leg was checked against `Code.string_to_quoted`'s tokenizer/parser-only pipeline. No untried axes.
- **Cycle-detection DoS** — verified; kept. The ruling rests on the dispositive compile-time/code-exec boundary (leg a, traced) with the polynomial bound (leg b, ran scaling + control) as secondary support; the boundary alone decides it, so no residual axis changes the outcome.
- **`expand_rescue` re-expansion loop** — remains **asserted**. I did not author a divergent macro to force the loop (it would hang the harness). The dismissal depends on `Macro.expand_once/2` single-step/fixpoint semantics and on this path being compile-time only; both are named in the `Depends on` line for the cross-file pass. Even if the loop is triggerable, it is a self-inflicted compile-time hang behind code execution, capping severity at nil-uplift.
- **`format_error` diagnostics** — verified; kept. I read all clauses (558-630) and confirmed every format string is a literal and every attacker-influenced value is an argument, not a directive. The only residual (how a downstream tool renders diagnostics) is out of this file and noted as a caveat, not a dismissed in-file sink.
- **Validation predicates, counter wrap, defensive clauses, absent sink classes** — verified; kept, each backed by a read and/or a run (`defs.exs`, grep) with empty `Not examined` lists.

