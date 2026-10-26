# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_quote.erl
<!-- sha256: 83772058f405963a31f9695756f25dabc813ca81f7e564c6316dbf4c2811b55e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T06:29:28Z_

I've completed a thorough review. Let me write up the report.

# Bug-hunting review — `elixir_quote.erl`

## Summary
After a full per-export pass of this quote/unquote/escape module, I found no correctness, crash, contract, or resource bug reachable from a documented public API entry point.

## Bugfixes

No findings.

The following spots were examined closely because they *looked* dangerous and turned out to be correct — recorded here so the reasoning is auditable:

- **`do_escape/2` improper-list path (`elixir_quote.erl:226-236`)** — the `try lists:reverse(T, [H]) of L -> do_quote_tail(L, Q) catch _:_ -> …` looks like an over-broad `catch _:_` that would swallow an `argument_error` thrown from `do_quote_tail` and then re-enter `reverse_improper/2` on a *proper* list (which returns a bare list, not a `{L,R}` tuple, and would `badmatch`). It is **not** a bug: in `try Expr of Pattern -> Body catch …`, only `Expr` is protected — the `of`-body (`do_quote_tail(...)`) runs outside the protected region, so a genuine escape error from `do_quote_tail` propagates uncaught. `reverse_improper/2` is therefore only ever entered with a truly improper `T`, where it returns a `{Acc, T}` tuple. Correct.

- **Struct custom-escape guard (`elixir_quote.erl:193-200`)** — `op /= {struct, Module}` (line 193) followed by `case op of {struct,_Module} -> argument_error` (line 197) is intentional: the line-193 check only excludes *self*-escaping during bootstrap (same module), while a `{struct, OtherModule}` op legitimately reaches the error branch ("custom escaping rules … not supported in struct defaults").

- **`do_quote_splice/4` buffer ordering (`elixir_quote.erl:504-517`)** — `[TH | Buffer]` prepends, which would reverse element order, but the input list was already reversed by `do_quote_tail`/`do_quote` via `lists:reverse(T, [H])` (line 439), so the double reversal restores original order. `[1, unquote_splicing(a), 2]` traces to `[1] ++ (a ++ [2])`. Correct.

- **`context = nil` into `build/7`** — `is_valid(context, C)` rejects `nil`, which would inject a runtime error. Confirmed at `elixir_expand.erl:243-246` that `Context` defaults to the module atom or `'Elixir'`, never `nil`; an explicit `context: nil` from the user is *meant* to be an error. Not a defect.

## Risks / caveats
- Several exports (`dot/5`, `tail_list/3`, `list/2`, `unquote/1`, `validate_runtime/2`) are "quote callbacks" that appear in generated code and are invoked at runtime with values the compiler produced. I reasoned about their contracts against the codegen in this same file (`do_runtime_list/3`, `do_quote_tail/2`, `do_quote_splice/4`); I did not execute generated modules. My confidence that `tail_list/3`'s second clause is live (improper-tail lists) rests on reading the codegen, not on a runtime trace.
- Cross-file claims are limited to `elixir_expand.erl:243-275` (read directly). Behaviour of `elixir_aliases:expand/4`, `elixir_dispatch:find_import(s)`, and `elixir_env:trace/2` was assumed to honour their documented return shapes; a shape drift there would surface in `do_quote`'s alias/import clauses, not in this file's logic.

### Ruled out

**Operator / precedence typo**
Reason: Every `,`/`;`/`orelse`/`andalso` in guards was read with explicit parentheses; the `?defs`/`?lexical` macros use `;` correctly in guard position, and `shallow_valid_elem/1` line 601 partitions as `is_atom ∧ is_list ∧ (is_atom ∨ is_list)` matching intent.
Axes examined: macro-expanded guard bodies, `==` vs `=:=` on atoms (line 98,110,242), mixed `,`/`orelse` (601) | Not examined: none.
Basis: verified.

**Missing clause / catch-all**
Reason: `dot/4` (548,558), `do_escape` (248), `do_quote` (441) all have terminal catch-all clauses; `try…of…catch _:_` at 228 is scoped to the reverse call only (verified above).
Axes examined: 3-tuples with non-atom head, improper lists, empty tuple in 413 guard, nil third element | Not examined: none.
Basis: verified.

**Wrong atom / wrong symbol**
Reason: Record field names, meta keys (`line`,`keep`,`counter`,`context`,`imports`,`ambiguous_op`,`stop_generated`,`column`,`generated`,`no_parens`,`delimiter`) and BIF names (`erlang:fun_info`, `code:ensure_loaded`, `function_exported`) all read consistently against their uses.
Axes examined: meta-key names across keystore/keydelete/keyfind, `__escape__`/`__struct__`/`__aliases__` atoms | Not examined: none.
Basis: verified.

**State / accumulator threading error**
Reason: `build/7` threads `Acc0..Acc3` monotonically; `do_tuple_linify/6` threads `NewGen`/`NewMeta` correctly; `do_quote_splice/4` threads `Buffer`/`Acc` (traced above).
Axes examined: Acc order in build, Gen flag flip on stop_generated, buffer/acc in splice | Not examined: none.
Basis: verified.

**Off-by-one / wrong arithmetic**
Reason: The only arithmetic is `QuoteLevel ± 1` in `has_unquotes` (symmetric increment on quote / decrement on unquote) and bitstring `bit_size rem 8` split, which reconstructs `<<Bits:Size, Bytes/binary>>` losslessly.
Axes examined: quote-level nesting depth 0/1/2, bitstring sizes 0/partial/byte-aligned | Not examined: none.
Basis: verified.

**Contract violation**
Reason: `build/7` returns the documented `{Q, VContext, Acc}`; `escape/3` reraises with pruned stacktrace preserving Kind/Reason; callbacks return AST or raise `ArgumentError` as documented.
Axes examined: build tuple arity, escape reraise shape, validate_runtime error shape | Not examined: none.
Basis: verified.

**Crash on stray / unexpected input**
Reason: `escape/3` wraps `do_quote` in try/catch; unescapable terms route through `bad_escape/1 → argument_error`; `do_quote` terminal clause (441) returns `Other` unchanged.
Axes examined: reference/port/anonymous-fun escape (bad_escape), improper list, bitstring, struct-without-module | Not examined: none.
Basis: verified.

**Resource leak on error path**
Reason: The module allocates no files, ETS, processes, monitors, or counters; it is pure AST transformation.
Axes examined: whole-file scan for ets/open/spawn/monitor | Not examined: none.
Basis: verified.

**Silent corruption**
Reason: List/splice ordering and map escaping (`lists:sort(maps:to_list)`) preserve semantics; `has_unquotes` treating nested-quote `bind_quoted`/`unquote:false` opts as disabling is intentional and does not affect actual expansion (unquotes in opts are still processed by `do_quote/2` line 339), only the `has_unquotes` pre-check gating a pattern-only error (`elixir_expand.erl:269`).
Axes examined: nested 2-arg quote with unquote in opts, map key ordering, splice mid-list | Not examined: none.
Basis: verified.

**Infinite loop / hang**
Reason: All recursion structurally decreases (list tails, tuple decomposition, quote-level counter with a base); no `receive`, no unbounded retry.
Axes examined: recursive descent on nested lists/tuples/quotes | Not examined: none.
Basis: verified.

**Quadratic-where-linear**
Reason: `do_quote_splice` prepends to `Buffer` then a single concat; `list/2`/`tail_list/3` do one `++` at runtime per splice site (unavoidable and non-nested).
Axes examined: buffer accumulation, per-splice concat | Not examined: none.
Basis: verified.

**Repeated work**
Reason: No compiled patterns/regex/BEAM scans; `code:ensure_loaded` is called once per struct-map escape, which is inherent.
Axes examined: per-call recompilation candidates | Not examined: none.
Basis: verified.

**Dead / unreachable code**
Reason: `meta/2`'s second clause (`do_quote(Meta,Q)` for non-quote op) is effectively unreachable because alias/import clauses requiring hygiene maps never match under escape ops, but it is a harmless fallback, not a bug; `tail_list/3` clause 2 is live for improper tails.
Axes examined: meta/2 non-quote reachability, tail_list clause coverage | Not examined: none.
Basis: verified.

**Documented behaviour mismatch**
Reason: `escape/3` doc ("lines kept, hygiene disabled") matches record `line=true`, `aliases_hygiene=nil`/`imports_hygiene=nil`; `linify/3` `line 0 → identity` matches doc.
Axes examined: escape record vs docstring, linify 0-line short-circuit | Not examined: none.
Basis: verified.

**Race condition / shared mutable state**
Reason: Pure functional module, no shared/process state.
Axes examined: whole-file scan for ets/process dict/system_flag | Not examined: none.
Basis: verified.

**Wrong scope / order of operations**
Reason: `annotate/2` deletes `counter` before `keystore(context)` where required (677); struct-op validation precedes `__escape__` invocation.
Axes examined: counter-delete vs context-store order, validate-before-call | Not examined: none.
Basis: verified.

**Encoding / unicode mishandling**
Reason: No byte/codepoint handling here; binaries built via `Elixir.Kernel:inspect` produce already-encoded messages; `atom_to_list`/`atom_to_binary` used only on internal atoms.
Axes examined: message-binary construction, atom_to_list on "Elixir." prefix | Not examined: none.
Basis: verified.

**Unclassified**
Reason: No novel/composite bug shape surfaced beyond the classes above.
Axes examined: struct custom-escape flow, quote-callback runtime contracts | Not examined: runtime execution of generated modules (reasoned from codegen only).
Basis: asserted.
Depends on: `elixir_expand.erl` codegen (`do_runtime_list`/`do_quote_splice` output shapes) and the runtime evaluator invoking `tail_list/3`,`list/2` with compiler-produced values — assumed to match the shapes constructed in this file.

### Did I verify this?

All classes except **Unclassified** are marked `verified`: I read the relevant clauses and tried every enumerated axis (the datatype axes for AST tuples/lists/bitstrings/maps and the quote-level counter), leaving `Not examined: none`.

**Unclassified** remains `asserted`: fully discharging it would require executing generated Elixir modules so that the runtime callbacks (`tail_list/3`, `list/2`, `dot/5`) receive real evaluator-produced arguments, which this read-only review cannot do. The `Depends on:` line names exactly where that verification must happen — the `elixir_expand.erl` codegen contract and the runtime evaluator. I traced those call sites statically (`elixir_expand.erl:272-282`, `do_runtime_list/3`, `do_quote_splice/4`) and found them consistent, but a runtime trace is the missing step, so I keep the entry `asserted` rather than overclaiming `verified`.

