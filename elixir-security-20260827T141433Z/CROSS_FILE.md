# Cross-file findings

## Asserted-dependency ledger

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| elixir.erl | elixir_errors.erl — `erl_warn/3` treats warning term as data, not format string | Confirmed | elixir_errors.erl.md ruled-out `~ts`/`characters_to_binary(Message)` renders `Message` as chardata data, not directives. |
| elixir.erl | elixir_parser (from .yrl) — assumes well-formed tokens; bounds own recursion | Confirmed | elixir_parser.yrl.md Risks + verified "Deep-nesting" ruled-out (O(input) yecc stack). |
| elixir.erl | elixir_tokenizer.erl — `tokenize/4` rejects bidi/newline/non-NFC/mixed-script/bad-escape | Confirmed | elixir_tokenizer.erl.md ruled-out entries (verified for strings/comments/escapes; asserted for NFC/mixed-script tables owned by String.Tokenizer). |
| elixir_aliases.erl | Elixir.Macro (macro.ex) — `inspect_atom` escapes atoms requiring quoting | Confirmed | lib-macro.ex.md verified `inspect_atom/3` quotes/escapes non-identifier atoms. |
| elixir_aliases.erl | module.ex — `Module.concat` documents unsafe atom creation, directs untrusted → `safe_concat` | Confirmed | lib-module.ex.md `CSCAN-module-concat-atom-exhaustion` verifies concat=`binary_to_atom`, safe_concat=`binary_to_existing_atom`. |
| elixir_aliases.erl | elixir_tokenizer.erl — identifiers in source already atomised there, bounding compile-path atom count | Confirmed | tokenizer is the sole sanitiser/atomiser of identifiers, bounded by source size (THREAT_MODEL invariant). |
| elixir_bitstring.erl | elixir_parser.yrl — bounds parsed-spec nesting | Confirmed | elixir_parser.yrl.md deep-nesting ruled-out (verified, O(input)). |
| elixir_bitstring.erl | Elixir.Macro:expand — fixpoint for non-recursive macros | Unresolved | lib-macro.ex.md addresses `expand/2` only for atom creation; fixpoint semantics not asserted. |
| elixir_bitstring.erl | threat model — compilation executes code by contract; macro exec not a defended boundary | Confirmed | THREAT_MODEL "out of scope: sandboxing evaluation … eval runs code." |
| elixir_bootstrap.erl | Elixir build system — bootstrap Kernel replacement only during self-compilation | Unresolved | Build-system invariant lives outside the reviewed folder; no report substantiates it. |
| elixir_clauses.erl | Elixir.Macro.expand_once/2 — single step, input unchanged at fixpoint | Unresolved | lib-macro.ex.md covers `expand_once/2` only for atom creation, not single-step/fixpoint semantics. |
| elixir_clauses.erl | compile-time-only reachability via elixir_expand | Confirmed | elixir_clauses.erl.md validation predicates verified compile-error-only, never a bypass. |
| elixir_code_server.erl | elixir_compiler.erl — `dispatch/4` returns compiler module even on crash path (no counter leak) | **Unresolved** | elixir_compiler.erl.md never verifies crash-safe return; `dispatch/4`/`optimize_defmodule` entries leave the `try/after` wrapping unexamined. |
| elixir_compiler.erl | elixir_config.erl / elixir_code_server.erl — config table seeded at app start; module pool server-minted | Confirmed | elixir_config.erl.md verifies `static/1`/`new/1` at boot; elixir_code_server.erl.md verifies server-minted `elixir_compiler_N` atoms. |
| elixir_compiler.erl | elixir_tokenizer.erl / elixir_interpolation.erl — `string_to_quoted!` rejects malformed UTF-8/bidi/NFC/mixed/escapes | Confirmed | Same tokenizer contract confirmed above (agent verification of tokenizer ruled-out set). |
| elixir_def.erl | elixir_code_server.erl (module-name uniqueness/serialisation) + elixir_module.erl (per-module tables) | Confirmed | THREAT_MODEL invariant (code_server single writer of `elixir_modules`); elixir_module.erl.md verifies per-module tables. |
| elixir_env.erl | elixir_tokenizer.erl — rejects bidi/NEL/newline, requires NFC in identifiers | Confirmed | tokenizer sanitiser contract confirmed; also negligible impact (unused-var warning). |
| elixir_erl.erl | Elixir.Inspect.Algebra — `format/2` cost scales with author-controlled compile-time signature | **Contradicted** | lib-inspect-algebra.ex.md `CSCAN-algebra-fits-quadratic-dos` (verified) shows `format/2` is O(N²) DoS-reachable through the documented API. |
| elixir_erl.erl | beam_lib / Code — safe decode of untrusted `.beam` chunks (`binary_to_term` upstream) | **Contradicted** | lib-code.ex.md `CSCAN-code-fetch-docs-unsafe-binary-to-term` (verified) shows Code decodes untrusted `.beam` with non-`:safe` `binary_to_term`. |
| elixir_erl.erl | elixir_def.erl — supplies Clauses/handlers from source under compilation | Confirmed | Compile-executes-code-by-design is in scope of caller trust (THREAT_MODEL). |
| elixir_erl.erl | elixir_erl_compiler.erl — compiles forms produced by expander/this file | Confirmed | Routing dependency; elixir_erl_compiler.erl.md reviews the compile step. |
| elixir_erl.erl | elixir_module.erl — owns/populates Set/Bag tables for module under compilation | Confirmed | elixir_module.erl.md verifies fresh per-module `DataSet`/`DataBag` in `build/4`. |
| elixir_erl.erl | elixir_parser/elixir_tokenizer — literal nesting depth bounded before code-gen | Confirmed | Parser deep-nesting verified O(input). |
| elixir_erl.erl | elixir_utils.erl:27-28 — `macro_name/1` prefixes existing atom; atom growth bounded | Confirmed | elixir_utils.erl.md verifies ≤1 `MACRO-` atom per existing atom, bounded by source. |
| elixir_erl_clauses.erl | elixir_erl_pass.erl — `translate/3`/`translate_args/3` return `{ErlForm, #elixir_erl{}}` | Confirmed | elixir_erl_pass.erl.md consistently threads `{Form, State}`; failure is caller self-crash only. |
| elixir_erl_clauses.erl | elixir_utils.erl — `extract_guards/1` returns `{Arg, Guards}` without atom creation | Confirmed | elixir_utils.erl.md lists it among verified pure helpers, no dynamic atom creation. |
| elixir_erl_clauses.erl | elixir_expand.erl / elixir_erl_pass.erl — clauses carry expander-normalised `{'->',Meta,[L,R]}` | Confirmed | Pass return-shape/clause-shape consistent; worst case is compile-time self-crash on trusted AST. |
| elixir_erl_compiler.erl | OTP compile/v3_core/beam_validator — diagnostic `Module` is pass name, not source-derived | Unresolved | OTP back-end is outside the folder; no report verifies it (report itself only asserts). |
| elixir_erl_compiler.erl | elixir_erl.erl / elixir_compiler.erl — `Opts` carry only `@compile`/env entries, not source atoms | Unresolved | elixir_compiler.erl.md never rules on `Opts`/`parse_transform` provenance. |
| elixir_erl_try.erl | elixir_erl.erl — `get_ann/1` tolerates/cleanly errors on out-of-range line metadata | Confirmed | elixir_erl.erl.md `get_ann/1` ruled-out (verified; integers guarded, no security effect). |
| elixir_fn.erl | elixir_dispatch.erl — `import_function`/`require_function` don't atomize untrusted text; no unbounded alloc | **Contradicted** | elixir_dispatch.erl.md rules the same sink out by assertion; its one verified axis shows `list_to_atom` grows the atom table on novel inputs; reachability Not examined. |
| elixir_fn.erl | elixir_expand.erl — `expand/3` terminates, no super-linear cost on module sub-expression | **Contradicted** | elixir_expand.erl.md Finding 1 (verified) measures ≈O(source²) amplification via `__ENV__`/`escape_map`. |
| elixir_fn.erl | elixir_tokenizer/elixir_parser — parser recurses ≥ same depth before `escape` sees AST | Confirmed | Parser builds AST by recursion first; deep-nesting verified O(input). |
| elixir_fn.erl | elixir_tokenizer.erl — `?bidi`/`?break` rejection; `Macro.to_string/1` returns well-formed chardata | Confirmed | tokenizer sanitiser confirmed; lib-macro.ex.md verifies `to_string/1` returns escaped binary, never bare atom. |
| elixir_errors.erl | elixir.erl — diagnostic `:file`/position from compiler/caller opts; tokenizer line ≤ newline count | Unresolved | elixir.erl.md never rules on file/line provenance or the line≤newline invariant. |
| elixir_errors.erl | elixir_tokenizer.erl — surfaces invalid UTF-8 as an error (message text stays valid chardata) | Confirmed | tokenizer `format_error` ruled-out (UTF8_DECODE_ERRORS_SURFACED); interpolation report verified `UnicodeConversionError` by execution. |
| elixir_expand.erl | elixir_aliases.erl — atom-exhaustion bounded (multi-MB single segment axis) | **Contradicted** | elixir_aliases.erl.md rules the `concat/1` sink out by assertion, "multi-MB single segment" Not examined; lib-module.ex.md verifies the same `binary_to_atom` sink is unbounded. |
| elixir_expand.erl | elixir_dispatch.erl — (import/dispatch atomization axes) | Contradicted | Same dispatch atom-growth axis ruled out by assertion (see elixir_fn/dispatch row). |
| elixir_expand.erl | elixir_clauses.erl / elixir_env.erl — scope-state cross-product only yields miscompiles | Confirmed | elixir_clauses.erl.md verified validation errors are compile errors, never bypass. |
| elixir_import.erl | (no concrete external contract; "Depends on for cross-file pass") | Unresolved | Report records only a placeholder; no specific invariant to resolve. |
| elixir_lexical.erl | Kernel.LexicalTracker — `set_file/2` stores name, does not open/stat/read path | Confirmed | lib-kernel-lexical_tracker.ex.md verified `set_file`/`reset_file` only store a string; no `File.*`/`Path.*` sink. |
| elixir_lexical.erl | Kernel.LexicalTracker — treats forwarded terms as opaque bounded state, not match-spec/callable | Confirmed | lib-kernel-lexical_tracker.ex.md verified no callable/match-spec sink. |
| elixir_module.erl | (no `Depends on` lines remain) | Unresolved | Report explicitly carries no outstanding external contract. |
| elixir_overridable.erl | elixir_code_server.erl (serialised defmodule, single writer) + elixir_module.erl (`data_tables/1` per-module) | Confirmed | code_server single-writer invariant + elixir_module.erl.md verified per-module tables. |
| elixir_parser.yrl | elixir.erl:533-534 — `put_parsing_state/1` sets `literal_encoder` from caller Opts | Confirmed | elixir.erl.md verified literal_encoder put (534) + parsing-state put/erase. |
| elixir_parser.yrl | elixir.erl:540-542,476-478 — parsing-state keys set before parse, erased after | Confirmed | elixir.erl.md verified set at `tokens_to_quoted` entry, erased in `after`. |
| elixir_parser.yrl | elixir_tokenizer.erl — emits only documented token tuple shapes | Confirmed | elixir_parser.yrl.md reduce-action edge-proofs verified against documented shapes. |
| elixir_quote.erl | (no concrete external contract; "Depends on for cross-file pass") | Unresolved | Report records only a placeholder; nothing concrete to resolve. |
| elixir_rewrite.erl | String/Enum/Path/Kernel/Macro stdlib — `join`/`inspect`/`to_string` return binaries | Confirmed | elixir_rewrite.erl.md frames `is_always_string` as correctness-only, no trust boundary (benign). |
| elixir_rewrite.erl | elixir_expand.erl / elixir_dispatch.erl — deliver well-formed AST (proper arg lists) | Confirmed | Worst case is self-inflicted `badarg` in caller's process; no boundary crossed. |
| elixir_sup.erl | (no `Depends on` line required) | Unresolved | Report records no external contract. |
| elixir_tokenizer.erl | elixir.erl — `string_to_tokens`/`string_to_quoted` convert binary→charlist raising on invalid UTF-8 | Unresolved | elixir.erl.md describes forwarding "raw bytes"; never rules on a conversion that raises on invalid UTF-8. |
| elixir_tokenizer.erl | elixir_interpolation.erl — `extract` `#{…}` recursion has no per-level multiplier | Confirmed | elixir_interpolation.erl.md verified (drove nesting to depth 20000; linear). |
| elixir_tokenizer.erl | unicode/tokenizer.ex (String.Tokenizer) — returns error for mixed-script/restricted/non-NFC | Confirmed | unicode-tokenizer.ex.md verified `{:error,{:mixed_script,…}}`/`{:error,{:unexpected_token,…}}` (caveat: that error path is quadratic). |
| elixir_tokenizer.erl | caller passing `static_atoms_encoder` honoring `{ok,_}\|{error,binary()}` | Unresolved | Explicit caller-contract; non-conforming return self-crashes caller (out of scope, addressed as a dependency not a defect). |
| elixir_tokenizer.erl | no in-file/parser path concatenates char-literal codepoints back into a reordering region | Confirmed | tokenizer report: char tokens become integer values, not source text. |
| elixir_utils.erl | elixir_module.erl / Mix — supply developer-derived `.beam`/source paths, not untrusted content | Confirmed | elixir_module.erl.md verified path built from validated module atom (developer-derived). |
| elixir_utils.erl | elixir_module.erl:585 — module-name→path join (`filename:join`) uses developer paths | Confirmed | elixir_module.erl.md verified `beam_location`/`validate_module_name` join of validated atom. |
| iex.erl | OTP net_kernel/distribution — remote-node connection authenticated by shared cookie | Unresolved | Documented Erlang distribution trust model; lives outside the folder, unverifiable here. |

## Findings

#### `CSCAN-crossfile-beam-chunk-unsafe-deserialization`
- **Finding ID** — `CSCAN-crossfile-beam-chunk-unsafe-deserialization`
- **Severity** — Medium
- **Confidence** — high
- **Files involved** — elixir_erl.erl.md, lib-code.ex.md
- **Issue** — `elixir_erl.erl` writes `{elixir_v1,Map,Specs}`/`ExCk`/docs chunks into `.beam` files and dismisses the reader-side risk with an *asserted* `Depends on: beam_lib/Code — responsible for safe decode of untrusted .beam files` (elixir_erl.erl.md ruled-out entries). The depended-on module does **not** honour that contract: lib-code.ex.md's verified finding `CSCAN-code-fetch-docs-unsafe-binary-to-term` (code.ex:2383, reached from the public `Code.fetch_docs/1`, path clause 2340) decodes an attacker-controlled `.beam`/`.chunk` with an unbounded, non-`:safe` `:erlang.binary_to_term/1` — reproduced as a 200 MB compression bomb and a 100 000-atom exhaustion from a single call. The two halves contradict: elixir_erl assumes safe decode; Code performs unsafe decode.
- **Resolution** — Track the `binary_to_term` fix in code.ex (bounded/`max_heap_size` decode) as the authoritative mitigation; confirm whether the `Dbgi`/`ExCk` chunk readers (`:beam_lib`, checker) share the same unsafe path, and if so extend the fix there rather than leaving it as an asserted cross-file dependency.

#### `CSCAN-crossfile-aliases-concat-atom-exhaustion`
- **Finding ID** — `CSCAN-crossfile-aliases-concat-atom-exhaustion`
- **Severity** — Medium
- **Confidence** — high
- **Files involved** — elixir_expand.erl.md, elixir_aliases.erl.md, lib-module.ex.md
- **Issue** — `elixir_expand.erl` dismisses alias atom-exhaustion with `Depends on: elixir_aliases.erl` (bounded atom growth), and `elixir_aliases.erl.md` in turn rules the `concat/1` sink out **by assertion**, explicitly leaving "multi-megabyte single-segment binary (large-atom cost)" as *Not examined*. But lib-module.ex.md's verified finding `CSCAN-module-concat-atom-exhaustion` reads `elixir_aliases.erl:175-176` and confirms `concat` calls `binary_to_atom` (permanent, unbounded atom creation), with `safe_concat` the only bounded variant. The "bounded" contract expand relies on is contradicted by a verified defect in the very function it depends on.
- **Resolution** — Adopt lib-module.ex.md's remediation (route untrusted input to `safe_concat`/document the unsafe path) as the fix of record for `elixir_aliases:concat/1`; upgrade elixir_aliases.erl's asserted dismissal to acknowledge the verified atom-exhaustion sink rather than treating it as bounded.

#### `CSCAN-crossfile-dispatch-atom-exhaustion`
- **Finding ID** — `CSCAN-crossfile-dispatch-atom-exhaustion`
- **Severity** — Medium
- **Confidence** — medium
- **Files involved** — elixir_fn.erl.md, elixir_rewrite.erl.md, elixir_dispatch.erl.md
- **Issue** — Both `elixir_fn.erl` and `elixir_rewrite.erl` dismiss their delegated lookups with `Depends on: elixir_dispatch.erl — import_function/require_function do not atomize untrusted text and return false/a resolved tuple without unbounded allocation.` `elixir_dispatch.erl.md` does not substantiate this: it *downgraded* the `find_imports_by_name`/`macro_name` atom-creation entries from verified to **asserted**, marks the "can call sites supply distinct novel names" reachability axis as *Not examined*, and its one verified axis shows the `list_to_atom` sink **does** grow the atom table on distinct novel inputs. The relied-on "no unbounded allocation" invariant is asserted, not verified, in the exact module cited.
- **Resolution** — Verify reachability directly in elixir_dispatch.erl source: determine whether attacker-influenced identifiers flow to the `list_to_atom`/`macro_name` sinks unbounded (vs. being pre-atomised by the tokenizer). If reachable, this is a genuine atom-exhaustion DoS to be fixed in elixir_dispatch; if not, upgrade the dispatch dismissal from asserted to verified so the fn/rewrite dependencies hold.

#### `CSCAN-crossfile-expand-superlinear-compile-cost`
- **Finding ID** — `CSCAN-crossfile-expand-superlinear-compile-cost`
- **Severity** — Low
- **Confidence** — medium
- **Files involved** — elixir_fn.erl.md, elixir_expand.erl.md
- **Issue** — `elixir_fn.erl` clears its module sub-expression path with `Depends on: elixir_expand.erl — expand/3 … does not itself introduce super-linear cost on the module sub-expression.` elixir_expand.erl.md's Finding 1 (verified) measures a ≈O(source²) compile-time amplification via `__ENV__`/`escape_map` (benchmark: 8× variables → ~24.6× time). The general "no super-linear cost" contract fn relies on is contradicted by a measured super-linear defect in expand. Whether fn's specific sub-expression drives the `escape_map` path is not established, hence medium confidence.
- **Resolution** — Confirm whether the `capture`/`fn` module sub-expression can reach the `__ENV__`/`escape_map` amplification (or any other super-linear expand path). If it can, fold it into the expand quadratic-cost finding; otherwise narrow elixir_fn's dependency to the specific expand paths it actually exercises.

#### `CSCAN-crossfile-inspect-algebra-compile-dos`
- **Finding ID** — `CSCAN-crossfile-inspect-algebra-compile-dos`
- **Severity** — Low
- **Confidence** — medium
- **Files involved** — elixir_erl.erl.md, lib-inspect-algebra.ex.md
- **Issue** — `elixir_erl.erl` dismisses inspect cost with `Depends on: Elixir.Inspect.Algebra — format/2 cost scales with the compile-time signature the author controls` (i.e. not attacker input). lib-inspect-algebra.ex.md reports the verified, High-reachability finding `CSCAN-algebra-fits-quadratic-dos`: `format/2` is O(N²) and DoS-reachable through the documented API. Under the folder threat model, the source being compiled (and thus the signature handed to `format/2`) is untrusted input, so the "author-controlled, bounded" assumption is contradicted by a verified quadratic-cost defect.
- **Resolution** — Determine whether elixir_erl's `format/2` call sites receive attacker-authored signatures at compile time; if so, this is a compile-time DoS amplifier and should reference the Inspect.Algebra quadratic fix. Otherwise, restate the dependency as "the specific signatures elixir_erl formats are small/compiler-generated," which the current report does not establish.

Note: the `elixir_code_server.erl → elixir_compiler.erl` compiler-module-leak dependency (row 13) is **Unresolved**, not a finding — elixir_compiler.erl's report simply never verified the crash-safe `return_compiler_module` path. Per the coverage-vs-defect rule it stays in the ledger; resolving it requires reading elixir_compiler.erl source to confirm whether `retrieve_compiler_module()` is wrapped crash-safely.
