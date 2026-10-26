# Cross-file findings

## Asserted-dependency ledger

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| elixir_rewrite.erl | `elixir/lib/**/*.ex` invoking `:elixir_rewrite.erl_to_ex/3` | Unresolved | No report substantiates a live `.ex` caller; the only `erl_to_ex` hit (lib-code-typespec) is a different fn `erl_to_ex_var/1`. Source already files low-conf `reverse-fun`. |
| elixir_rewrite.erl | OTP `erlang`/`maps`/`binary`/`os` BIF names stable | Unresolved | External OTP invariant; no in-set report covers it. |
| elixir_utils.erl | `kernel/file` returns only `{ok,#file_info{}}`/`{error,_}` | Unresolved | External OTP module; no report. (The sibling race-condition dep at line 124 was upgraded to verified in-repo, so it is no longer a dependency.) |
| elixir_module.erl | `elixir_def`/`Module` bag attrs have `{Key,Funs}`/`{Name,Arity}` shapes validators match | Unresolved | elixir_def.md verifies only `{def,_}` 6-tuple rows, not the accumulate-bag validator shapes; lib-module.ex.md covers runtime get/put only. |
| elixir_module.erl | process in `elixir_compiler_info` always sends `{Ref,ack}` or dies | Unresolved | elixir_compiler.erl.md never addresses `elixir_compiler_info` or the `{Ref,ack}` handshake. |
| elixir_module.erl | `{defmodule,Module,self(),Tuple}` atomically detects in-progress dup | Confirmed | elixir_code_server.md "Wrong scope/order" (verified): lookup-emptiness check before insert, serialized via single gen_server ("Race condition", verified). |
| elixir_module.erl | `elixir_utils:characters_to_list/1` converts chardata→codepoints | Confirmed | elixir_utils.md "Encoding/unicode" (verified): passes valid list/binary, raises on `{error,_}`/`{incomplete,_}`. |
| elixir_erl_var.erl | `elixir.hrl` defines `#elixir_erl`/`#elixir_ex` fields & `?ann`/`?var_context` | Unresolved | No report exists for `elixir.hrl` (header not in review set). |
| elixir_erl_var.erl | `elixir_erl_pass`/`elixir_erl_for`/`elixir_erl_try` consume returned tuple shapes | Confirmed (partial) | elixir_erl_for.md "Verified callee-shape deps" (verified) confirms `build/2`→`{AtomName,Scope}`; erl_pass leans on it but "not re-verified inside source"; erl_try does not mention erl_var. |
| elixir_erl_compiler.erl | OTP `sys_core_fold`/`v3_core`/`erl_lint` warning atoms stable | Unresolved | External OTP invariant; no report. |
| elixir_tokenizer.erl | `elixir_interpolation:extract/6` threads warnings/cursor through reentry; `\n`-prefixed first heredoc part | Confirmed | interpolation.md "State/accumulator threading" + "Contract violation" (both verified), heredoc `\n`-prepend under verified Encoding. |
| elixir_expand.erl | env/`#elixir_ex` keys `context_modules`/`lexical_tracker`/`runtime_modules` match | Unresolved | elixir_env.md "Wrong atom" (verified) lists only `prematch/unused/vars/version` + other map keys; the three named keys are not addressed. |
| elixir_expand.erl | parser/quote produce 3-elem `{Form,Meta,Args}` nodes, list Meta | Confirmed | elixir_parser.md + elixir_quote.md "Contract violation" (both verified) confirm emitted AST tuple shapes. |
| elixir_expand.erl | `elixir_clauses`/`elixir_dispatch` raise `invalid_clauses`/`nested_comparison` via `?MODULE` | Unresolved | Neither target report mentions these tags (only elixir_expand.erl.md contains them). |
| elixir_aliases.erl | `lib/module.ex` (concat crash-on-charlist / documented-behaviour mismatch) | Unresolved | lib-module.ex.md confirms `concat/1,2` doctests but never addresses the atom\|binary-vs-charlist input contract the finding rests on. |
| elixir_parser.yrl | `elixir_quote`/`elixir_expand` consume emitted meta keys/tags, unchanged | Unresolved | Both target reports verify only their own internal key self-consistency, not the parser as producer. |
| elixir_parser.yrl | `elixir_tokenizer` produces only well-formed token tuples | Confirmed | elixir_tokenizer.md "Contract violation" + "Wrong atom" (both verified): documented success/token tuple shapes. |
| elixir_parser.yrl | generated `elixir_parser.erl` exercises each action production | Unresolved | Generated file has no report in the set. |
| elixir_quote.erl | `elixir_expand` codegen (`do_runtime_list`/`do_quote_splice`) & runtime invoke `tail_list/3`,`list/2` with compiler values | Unresolved | elixir_expand.md never addresses quote runtime callbacks or these codegen output shapes. |
| elixir_overridable.erl | `data_tables/1`→`{Set,Bag}`; overridable row `{{overridable,Tuple},Count,{DefRow,Clauses},false}`; def rows 6-tuples | Confirmed (partial) | elixir_def.md "Contract violation"+"Crash on stray input" (verified): 6-tuple `{def,_}` row + `store/5` destructuring match; the `{Set,Bag}`/row-layout halves not explicitly ruled on. |
| elixir_overridable.erl | `elixir_expand` consumes `{FinalKind,FinalName,Meta}` from super path | Unresolved | elixir_expand.md touches super only via arity match/threading; never verifies consumption of this return tuple. |
| elixir_overridable.erl | `elixir_module` compiles a module in one process owning `{Set,Bag}` | Confirmed (asserted-level) | elixir_module.md "Race condition" affirms per-module process-owned tables — but Basis asserted (itself defers to code_server), not verified. |
| elixir_def.erl | `elixir_module` guarantees one compile process per module owns `{Set,Bag}` | Confirmed (asserted-level) | Same elixir_module.md "Race condition" entry; asserted, not verified. |
| elixir_import.erl | `data_tables/1` per-module keying ⇒ concurrent `record/4` last-writer-wins | Unresolved | elixir_module.md covers table ownership/serialization but never `record/4` import inserts or same-`{import,Tuple}` semantics. |
| elixir_import.erl | sole caller chain expand.erl:157 ⇒ `InfoCallback` always module `__info__/1` | Unresolved | elixir_expand.md does not confirm the import/6 call site, its sole-caller status, or the fixed `Ref:'__info__'/1` callback. |
| iex.erl | OTP `user_drv` `initial_shell` accepts `{M,F,A}`/`{remote,Node,{M,F,A}}` | Unresolved | External OTP module; no report. |
| elixir_bootstrap.erl | `elixir_expand`/parser supply `__CALLER__` as `{Line,S,Env}`, Env map with `module`+`line` | Unresolved | expand.md covers `__ENV__` field access, not the `__CALLER__` triple; parser.md silent. |
| elixir_bootstrap.erl | `Elixir.Kernel` (once loaded) exports `'MACRO-@'/2` | Confirmed | lib-kernel.ex.md: `@` macro manually reviewed correct, file byte-identical to upstream. |
| elixir_erl_try.erl | `elixir_erl_pass` (sole caller, line 155) calls `clauses/3` only on rescue/catch of expanded try | Unresolved | erl_pass.md only generically states it "operates on macro-expanded AST"; never addresses this call site/specificity. |
| elixir_code_server.erl | out-of-folder `Kernel.ParallelCompiler`/`Code` are acquire/release lock callers guaranteeing lock release | Unresolved | Verified directly: neither lib-kernel-parallel_compiler.ex.md nor lib-code.ex.md addresses the acquire/release lock protocol. (Source self-corrected the earlier mis-attribution to elixir_compiler/elixir_module.) |
| elixir_module.erl / elixir_overridable.erl / elixir_def.erl / elixir_expand.erl | env/meta keys minted by `elixir_env`/`elixir_aliases` match names read | Unresolved | Same as the elixir_expand env-keys row; the originating dismissals remain asserted. |
| elixir_errors.erl | `Module:format_error/1` (silent-corruption/encoding dismissals) | Confirmed (self-resolved) | Source upgraded both dismissals to `verified` by executing the malformed-UTF-8 axis (→`badarg`); no outstanding dependency. |
| elixir_lexical.erl | `'Elixir.Kernel.LexicalTracker'` contract (claimed absent from checkout) | **Contradicted** | lib-kernel-lexical_tracker.ex.md describes exactly that module as "a faithful gen_server implementation" with the referenced API — the module IS in the checkout. See finding below. |

## Findings

#### `CSCAN-crossfile-lexicaltracker-exists`

* **Finding ID** — `CSCAN-crossfile-lexicaltracker-exists`
* **Severity** — Low
* **Confidence** — high (both reports read; the contradiction is named at specific lines)
* **Files involved** — `elixir_lexical.erl.md`, `lib-kernel-lexical_tracker.ex.md`
* **Issue** — `elixir_lexical.erl.md:24` files three low-confidence findings and justifies their confidence level with the assertion that their basis is "an un-inspectable cross-module contract on `'Elixir.Kernel.LexicalTracker'`," stating that module "does not exist anywhere in this checkout" (the author appears to have grepped for the BEAM-mangled atom rather than the Elixir source module). But `lib-kernel-lexical_tracker.ex.md` exists in the same review set and describes precisely that module — `Kernel.LexicalTracker` at `lib/kernel/lexical_tracker.ex` — as "a faithful gen_server implementation of the lexical tracker," substantiating the `add_reference`/`warn_require`/`warn_alias`/`warn_import` and `collect_unused_*`/cache/stop API surface. The premise on which `elixir_lexical`'s findings were held at low confidence (contract not inspectable) is therefore false: the contract *is* inspectable and is substantiated by a peer report.
* **Resolution** — Re-evaluate the three `elixir_lexical.erl` findings against `lib-kernel-lexical_tracker.ex.md` (and the actual `lib/kernel/lexical_tracker.ex` source): each can now be either verified or dismissed against the real API rather than left at low confidence on a "module absent" rationale. Note this is a correction to the *confidence basis* of the originating findings, not a newly discovered code defect.

**Note on the Unresolved rows.** A recurring, non-defect pattern runs through the compiler-state modules: `elixir_module` ↔ `elixir_def` ↔ `elixir_overridable` ↔ `elixir_import` dismiss race/shape concerns by deferring to a single-compile-process-owns-`{Set,Bag}` invariant, but the one report that would anchor it (`elixir_module.erl.md` "Race condition") is itself only *asserted* and defers further to `elixir_code_server`. No link in that chain is `verified`. Likewise, several out-of-folder/OTP/generated-file/header dependencies (`kernel/file`, OTP BIF & warning atoms, `user_drv`, generated `elixir_parser.erl`, `elixir.hrl`) have no report to resolve against. These are coverage gaps recorded above, not defects — they are intentionally left as ledger rows and not minted as findings.
