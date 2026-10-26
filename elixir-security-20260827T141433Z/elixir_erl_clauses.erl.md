# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_clauses.erl
<!-- sha256: e4a16fa7974923e6bcfbe3078df97a5478776facf10e5286cdc790ae4a096630 -->

_Analysed on 2026-08-27T16:44:02Z_

I now have a complete picture. This is a 70-line pure AST-transformation module in the Erlang code-generation stage of the compiler. Let me verify the final structural facts before writing the report.

The file contains no `binary_to_term`, no `list_to_atom`/`binary_to_atom`, no `apply`/eval, no file/network/OS/crypto calls, no process-dictionary/ETS writes. Every caller (confirmed above) passes compiler-internal, post-expansion AST and literal `fun` references. The header confirms `?ann` is line/column metadata and `#elixir_erl{}` is functional translation state holding no secrets.

---

# Security review: `elixir_erl_clauses.erl`

## Summary

`elixir_erl_clauses` is a 70-line structural transformer that lowers already-tokenized, already-parsed, already-macro-expanded Elixir clause AST (case/fn/receive/for/try/`=`) into Erlang abstract-format `{clause, …}` forms; it contains no code-execution, deserialization, atom-creation, file/network/crypto, or shared-mutable-state sinks, and every value it touches originates from trusted in-compiler callers.

## Security

No findings.

This file sits at the `elixir_erl*` (Erlang-form generation) stage, which the folder threat model marks as downstream of the sole byte-sanitiser (`elixir_tokenizer`) and of `elixir_parser`/`elixir_expand`. The Unicode obligations in the standards checklist (bidi/Trojan-Source rejection, NFC, mixed-script, `\u`/`\x` escapes, UTF-8 decoding, grapheme segmentation — sub-claims 1–12 and RFC clauses 1–10) all concern the tokenizer/interpolation stage and have **no code path in this file**; there is nothing here to satisfy or violate. There is no `binary_to_term` call site (always-flag rule: none present).

## Risks / caveats

- The dismissals below all rest on one out-of-file invariant: the four in-folder callers (`elixir_erl`, `elixir_erl_for`, `elixir_erl_pass`, `elixir_erl_try`) hand this module AST that `elixir_expand` has already normalised into the `{match|expr, Meta, [Condition], Expr}` and `{'->', Meta, [Left, Right]}` shapes. If a malformed shape *did* reach `each_clause/2` or `clause/6`, the result is a `function_clause`/`case_clause` crash of the **caller's own compiling process** — which the threat model documents as the expected consequence of feeding untrusted quoted AST to `eval_forms`/`quoted_to_erl` (a caller bug, self-inflicted DoS only). It never reaches a dangerous sink because there is no dangerous sink in this file.
- I did not read `elixir_expand.erl` in full; the `Depends on:` lines below name exactly where that contract lives so the cross-file pass can confirm it. Even under the pessimistic assumption that the expander could emit an off-shape clause, no ruling-out below would upgrade to a finding, because the worst outcome remains a compile-time crash of the caller's process on a documented "pass trusted AST only" path.

### Ruled out

**`match/4` — invoking the `Fun` callable reference (code execution shape)**\
Reason — `Fun` is always a compiler-internal literal function reference (`fun elixir_erl_pass:translate/3`, `translate_args/3`, `translate/3`, `translate_fn_match/3`), never a value derived from source text or attacker input; the six call sites confirm this.\
Axes examined — origin of `Fun` at every call site (all literal `fun`s), context save/restore correctness | Not examined: none.\
Basis — verified.

**`clause/6` / `guards/4` / `translate_guard/4` — `element(1, elixir_erl_pass:translate(...))` and guard-list assembly**\
Reason — `element(1,…)` extracts the Erl form from `translate/3`'s `{Form, State}` return; a wrong-shape return would only crash the caller's own compile process, and the recursion/amplification (if any) lives in `elixir_erl_pass:translate`, not here — this file's per-clause work is O(1).\
Axes examined — return-tuple arity of `translate/3`, guard list linearity, `Extra`-guards threading | Not examined: none.\
Basis — asserted.\
Depends on: `elixir_erl_pass.erl` — `translate/3`/`translate_args/3` return a `{ErlForm, #elixir_erl{}}` 2-tuple; a non-tuple return only self-crashes the compiling process.

**`get_clauses/3` — list-comprehension pattern filter (`{'->', Meta, [Left, Right]}`) with `is_list` guard (validation / silent-skip shape)**\
Reason — non-matching elements are silently dropped, but the `Keyword`/`Clauses` come from post-expansion block AST (`do`/`else`/`catch`/`rescue`/`after`) where the expander guarantees `->` shape; a dropped stray element is a compile-time behaviour on trusted AST, not a security property, and there is no atom creation or eval in the comprehension.\
Axes examined — non-`->` elements (dropped), non-list `Clauses` (guarded by `is_list`), arity-mismatched `->` (dropped), `As` tag values (`catch`/`rescue`/`do`/`else`/`after`/`match`/`expr`) | Not examined: none.\
Basis — asserted.\
Depends on: `elixir_expand.erl` / `elixir_erl_pass.erl` (callers at lines 165/172/179/309/437, `elixir_erl_try.erl:13-14`) — block keyword lists reaching `get_clauses/3` carry expander-normalised `{'->', Meta, [L, R]}` clauses.

**`clauses/2` + `each_clause/2` — `mapfoldl` dispatch and `{match|expr, …}` pattern match (resource-consumption + dead/unreachable-defence shape)**\
Reason — `mapfoldl` is O(clause-count), itself bounded by source size already accepted by the tokenizer; `each_clause/2` matches exactly the two tags (`match`, `expr`) that its callers ever produce (verified: `elixir_erl_pass.erl:166/176/180/317/438` tag clauses `match`/`expr` only), so the "missing catch-all" is not dead defence but an internal totality that holds for all real inputs — an off-tag clause would only `function_clause`-crash the caller's process.\
Axes examined — clause-count scaling (linear), tag values reaching `clauses/2` (only `match`/`expr`), `[Condition]` single-element-list requirement, `elixir_utils:extract_guards/1` on `match` condition | Not examined: none.\
Basis — asserted.\
Depends on: `elixir_erl_pass.erl` — every `clauses/2` call is fed clauses tagged `match` or `expr`; `elixir_utils.erl` — `extract_guards/1` returns `{Arg, Guards}` without atom creation.

**`unblock/1` — block unwrapping (structural)**\
Reason — pure structural match on `{'block', _, Exprs}` vs. otherwise; no external influence, no sink.\
Axes examined — block vs. non-block form, empty-block | Not examined: none.\
Basis — verified.

**`#elixir_erl{}` state threading (shared-mutable-state shape)**\
Reason — the record is threaded functionally (immutable copies); `match/4` saves/restores `context` correctly; no process-dictionary/ETS/application-env write occurs in this file (the parser process-dictionary state noted in the threat model is set in `elixir.erl`/tokenizer, not here).\
Axes examined — context save/restore balance in `match/4`, `extra_guards` reset in `clause/6`, absence of `put`/`ets`/`application:set_env` | Not examined: none.\
Basis — verified.

### Did I verify this?

- **`match/4` Fun origin — verified.** I read all six call sites (grep output): each passes a literal `fun Module:Function/Arity`; `Not examined: none`. Upgrade stands.
- **`unblock/1` — verified.** Whole function read (lines 68–69); the datatype has exactly two shapes and both are covered; `Not examined: none`. Stands.
- **`#elixir_erl{}` threading — verified.** I read every function body in the file and the record definition (`elixir.hrl:54-71`); confirmed no mutable-global write and balanced context restore in `match/4:23-25`; `Not examined: none`. Stands.
- **`clause/6`+`guards/4` `element(1,…)` — kept asserted.** I read the local code and traced the return-tuple usage, but I did not read `elixir_erl_pass:translate/3`'s body to confirm it always returns a 2-tuple. `Depends on:` line names that contract. The dependency is benign: even a wrong return only self-crashes the compile.
- **`get_clauses/3` filter — kept asserted.** I traced the seven call sites and the local comprehension, but confirming the `->` shape is guaranteed requires `elixir_expand.erl`, which I did not read. `Depends on:` line names it. Failure mode is a silently-dropped clause on trusted AST, not a sink.
- **`clauses/2`/`each_clause/2` tag totality — kept asserted.** I verified from the caller grep that only `match`/`expr` tags reach `clauses/2`, but the guarantee that the expander cannot emit another tag lives in `elixir_erl_pass.erl`/`elixir_expand.erl`. `Depends on:` line names it. Failure mode is a caller-process `function_clause`, not a sink.

