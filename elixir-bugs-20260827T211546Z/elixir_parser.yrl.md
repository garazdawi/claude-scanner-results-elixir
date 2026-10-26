# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_parser.yrl
<!-- sha256: d040825a21dd29c0a2a109ee09371cef644a216fdbc2f4f0593a248f79108702 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T06:25:41Z_

# Bug-hunting review — `elixir_parser.yrl`

## Summary
No genuine correctness/crash/contract bugs were found reachable through the documented public API (`elixir_parser:parse/1`, driven by `elixir:string_to_quoted/5` and friends); the grammar and its Erlang action helpers are internally consistent with their callers.

## Bugfixes

No findings.

Notes from the two claims I most suspected, both cleared:

- **`warn_no_parens_after_do_op/1` (line 1299–1306) passes a bare integer `Line` to `warn/2`, while every other warning passes `{Line, Column}`.** This is *not* a bug: parser warnings are drained by `elixir.erl:emit_warnings/3` into `elixir_errors:erl_warn(L, File, M)`, whose `-spec` (`elixir_errors.erl:19`) declares `location() :: non_neg_integer() | {non_neg_integer(), non_neg_integer()}`. Both shapes are valid; the bare integer is the same shape `erl_warn(0, ...)` uses internally. Verified by reading `elixir_errors.erl:245-249`.
- **`access_expr -> char` (line 283) swaps the `Number`/`Original` argument order relative to `int`/`flt` (lines 281–282).** This is *not* a bug: for `int`/`flt` the numeric value is in `element(3, Token)` (via `number_value/1` reaching `element(2)`’s value slot) and the original text elsewhere; for `char` tokens the two slots are transposed, so the swapped call restores the same `handle_number(NumericValue, Token, OriginalText)` contract. `handle_number/3` (lines 722–726) treats both uniformly.

## Risks / caveats
- This is a yecc grammar file; its only documented public entry point is the generated `elixir_parser:parse/1`, invoked with a token stream the tokenizer produced. The Erlang action helpers (`build_op`, `build_call`, `collect_stab`, etc.) are never called with arbitrary caller-supplied terms — they only see node shapes the grammar rules construct. Several helpers (`extract_identifier/1` line 908, `number_value/1` line 728, `build_dot_alias/3` line 874) are deliberately partial and would `function_clause` on foreign shapes, but no grammar production can deliver such a shape, so they are not reachable-from-public-API bugs.
- Two cross-file assumptions underpin the "no findings" conclusion; if either is false, a ruled-out class could become a finding:
  - **`elixir.erl:put_parsing_state/1` (line 531–542) always initializes `elixir_parser_warnings`, `elixir_parser_columns`, `elixir_token_metadata`, `elixir_literal_encoder` before `parse/1` runs.** Verified by reading `elixir.erl:464-479, 531-542`. If a caller invoked `elixir_parser:parse/1` directly without this setup, `warn/2` (line 1349) would prepend to `undefined` (improper list) and `?columns()`/`?token_metadata()` would misread — but that is undocumented-internal misuse, out of scope.
  - **`elixir_errors:erl_warn/3` accepts both `integer()` and `{integer(),integer()}` locations.** Verified (`elixir_errors.erl:19,245-249`).

### Ruled out

**Operator / precedence typo**
Reason: The `Left/Right/Nonassoc` precedence table (lines 69–97) matches the documented operator table it cross-references, and guard/boolean expressions in actions (`?rearrange_uop`, `newlines/2` `is_integer(Count) and (Count > 0)`) use total, non-crashing forms.
Axes examined: `and` vs `andalso` in `newlines/2` guard, `//`/`..` relative precedence (`1..9//2`), `not in` 4-tuple op token, unary `//` split | Not examined: none.
Basis: verified.

**Missing clause / catch-all**
Reason: Partial action functions (`extract_identifier/1`, `number_value/1`, `build_dot_alias/3`, `check_stab/2`) are each fed only the finite set of node shapes their grammar productions build; `build_op/3` covers `//`, rearranged-unary `in`, `not in`, and a general fallback.
Axes examined: `//` non-range LHS (error path line 746), `not`/`!` `in` rearrange, alias-after-atom, char/int/flt token shapes | Not examined: none.
Basis: verified.

**Wrong atom / wrong symbol**
Reason: Atom keys emitted (`'__block__'`, `'..//'`, `'<<>>'`, `'%{}'`, `'__aliases__'`, `no_parens`, `from_brackets`, `end_of_expression`) match the AST/meta contract consumed downstream; no drifted field names.
Axes examined: meta keys in `annotate_eoe`, `build_access`, `build_sigil`; binary_to_atom op selection (line 1046) | Not examined: downstream consumers' exact key expectations (cross-file).
Basis: asserted.
Depends on: `elixir_quote.erl`/`elixir_expand.erl` — consume the meta keys and node tags this parser emits; assumed unchanged.

**State / accumulator threading error**
Reason: List accumulators are built reversed and flipped once via `reverse/1,2` at the boundary (e.g. `call_args_no_parens_comma_expr`, `container_args`, `kw_base`→`kw_call`/`kw_data`); `collect_stab/3` correctly resets the `Exprs` accumulator to `[]` after each `->`.
Axes examined: reversed kw lists (lines 576–582, 588–589), `collect_stab` reset (line 1132), `unwrap_when`/`unwrap_splice` | Not examined: none.
Basis: verified.

**Off-by-one / wrong arithmetic**
Reason: The only literal offset is `build_unary_op` for `//` (`Column+1` for the outer `/`, `Column` for the inner, line 769), which correctly places the two synthesized `/` nodes; empty-grammar location is `{1, 1, nil}` (lines 101, 106) as intended.
Axes examined: `//` column split, empty-input line/column, `next_is_eol` count propagation | Not examined: none.
Basis: verified.

**Contract violation**
Reason: All action clauses return the AST/tuple shapes their grammar nonterminal is consumed as (`{List, Location}` from `build_list`, `{Meta, Args}` pairs from `call_args_parens`, `{'->', ...}` stab nodes); errors go through `return_error/3` which yecc surfaces as `{error, {Line, _, [Msg, Token]}}`, exactly the shape `elixir.erl:470-473` matches.
Axes examined: `return_error` tuple shape vs `tokens_to_quoted` matcher, `build_list` 2-tuple, `call_args_parens` meta pair | Not examined: none.
Basis: verified.

**Crash on stray / unexpected input**
Reason: Untrusted source reaches `parse/1` only as a tokenizer-validated token stream; malformed *syntax* yields yecc `{error, ...}`, not a crash. No action clause pattern-matches on input the grammar cannot produce.
Axes examined: empty input, lone `->`, empty `()`, trailing comma, `%{}` update forms, sigil/heredoc token shapes | Not examined: adversarial token streams bypassing the tokenizer (out of scope — undocumented internal use).
Basis: asserted.
Depends on: `elixir_tokenizer.erl` — produces only well-formed token tuples of the shapes these productions match.

**Resource leak on error path**
Reason: No files, ETS, processes, monitors, or timers are created here; only process-dictionary parser state, which `tokens_to_quoted/3` unconditionally `erase`s in an `after` block (`elixir.erl:474-478`).
Axes examined: `return_error` mid-parse throw, `string_tokens_parse` throw (line 1098) | Not examined: none.
Basis: verified.

**Silent corruption**
Reason: `append_non_empty/2`, `build_block/2`, and `collect_stab/3` preserve/return their accumulators on every clause including empty ones; block wrapping (`build_block([Expr],_) -> Expr`) matches the documented single-expression-unwrap behaviour.
Axes examined: empty args (`append_non_empty(L,[])`), single-expr block, `unquote_splicing` block special-case (line 802) | Not examined: none.
Basis: verified.

**Infinite loop / hang**
Reason: All action recursion (`check_stab/2`, `collect_stab/3`, `charlist_parts`, `string_parts`) walks a finite, strictly-shrinking list; yecc's driver is not modified. No `receive`/gen_server here.
Axes examined: `check_stab` list recursion, `collect_stab` termination on `[]` | Not examined: none.
Basis: verified.

**Quadratic-where-linear**
Reason: The one `++`-in-recursion candidate is `list_args -> ... reverse('$1','$3')` (line 596) and `Start ++ End` in `unwrap_when` (line 1155), both single applications on the reduce boundary, not per-iteration loops; comma-list accumulators use `[H|Acc]` + one final `reverse`.
Axes examined: `kw_base`/`container_args_base` accumulation, `append_non_empty` | Not examined: none.
Basis: verified.

**Repeated work**
Reason: No pattern compilation, BEAM scanning, or per-call rebuilding; `?columns()`/`?token_metadata()` are single process-dictionary reads.
Axes examined: macro-expanded `get/1` reads | Not examined: none.
Basis: verified.

**Dead / unreachable code**
Reason: `Expect 3.` documents the known shift/reduce conflicts; every action clause is reachable from a production. No obvious leftover asserts.
Axes examined: `warn_pipe/2` fallback, `with_assoc_meta/2` fallback, `build_dot_alias` atom clause | Not examined: whether all fallback clauses are actually reachable (would need yecc state analysis).
Basis: asserted.
Depends on: generated `elixir_parser.erl` state machine — assumed to exercise each action production per the `.yrl` rules.

**Documented behaviour mismatch**
Reason: Header comments describe operator precedence and the no-parens ambiguity rules; the productions (`no_parens_one_ambig_expr`, `call_args_no_parens_*`) implement exactly the described `f g a, b` ⇒ `f(g(a,b))` interpretation.
Axes examined: ambiguity examples in lines 116–154 vs productions 252–256, `//`-must-follow-`..` error message vs `build_op` | Not examined: none.
Basis: verified.

**Race condition / shared mutable state**
Reason: Parser state lives entirely in the calling process's process dictionary, set up and torn down per `tokens_to_quoted/3` call; no cross-process sharing.
Axes examined: process-dictionary keys lifecycle | Not examined: none.
Basis: verified.

**Wrong scope / order of operations**
Reason: Meta is captured from the correct tokens before construction (e.g. `build_nested_parens` filters line/column from the target node; `do_end_meta` uses the matching `do`/`end` tokens).
Axes examined: `build_nested_parens` location filter, `annotate_eoe` stab-vs-nonstab ordering | Not examined: none.
Basis: verified.

**Encoding / unicode mishandling**
Reason: String/charlist building routes through `elixir_utils:characters_to_list/1` / `characters_to_binary/1` with a `try` catching `UnicodeConversionError` and converting to `return_error` (lines 1017–1024); atoms use `binary_to_atom(_, utf8)`.
Axes examined: charlist heredoc conversion error path, quoted-atom utf8 | Not examined: none.
Basis: verified.

### Did I verify this?

- **Operator/precedence, Missing-clause, State-threading, Off-by-one, Contract, Resource-leak, Silent-corruption, Infinite-loop, Quadratic, Repeated-work, Documented-behaviour, Race, Wrong-scope, Encoding** — kept `verified`: I read the cited action clauses and traced each listed axis to its result; `Not examined: none` in each.
- **Wrong atom / wrong symbol** — remains `asserted`. Re-reading this file confirms the atoms emitted are self-consistent, but whether every meta-key/tag matches what `elixir_quote`/`elixir_expand` read is a cross-file fact this file cannot settle. `Depends on:` line retained; the cross-file pass should confirm against those modules' reports.
- **Crash on stray input** — remains `asserted` for the adversarial-token-stream axis, which is only reachable by bypassing the tokenizer (undocumented internal misuse, explicitly out of scope). `Depends on: elixir_tokenizer.erl` retained.
- **Dead / unreachable code** — remains `asserted`: proving each fallback clause reachable requires analyzing the generated LALR state table, which I did not do. `Depends on:` the generated `elixir_parser.erl` retained.

