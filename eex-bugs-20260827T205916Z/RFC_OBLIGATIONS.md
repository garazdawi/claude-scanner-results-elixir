<!-- rfc-governed: no -->
<!-- standards: -->

# Documented-contract / behavioural-obligation list — `elixir/lib/eex`

## Summary of what this folder is

This folder is **EEx** — *Embedded Elixir*, Elixir's standard-library templating engine (`.ex` sources: `eex.ex`, `eex/compiler.ex`, `eex/engine.ex`, `eex/smart_engine.ex`). It tokenizes template strings containing `<% %>` / `<%= %>` / `<%% %>` / `<%!-- --%>` tags, compiles them into an Elixir AST via a pluggable *engine* behaviour, and evaluates or defines functions from templates.

There is **no network protocol, wire format, or cryptographic algorithm** here. The correctness of this code is governed entirely by:

* the **EEx engine behaviour** (`EEx.Engine` callbacks) — an internal Elixir contract;
* the documented public API of the `EEx` module (`eval_string/3`, `eval_file/3`, `compile_string/2`, `function_from_string/5`, etc.), including its `@doc`/`@spec` return shapes and its documented exception (`EEx.SyntaxError`);
* general secure-coding practice (the smart engine's job is to build correct AST; a templating engine that emits user data must not, itself, be a source of injection — but that's a secure-coding concern, already covered by the per-file security review, not a protocol standard).

None of these are external protocol/algorithm standards, so the RFC-governance verdict is **no** and the external-standards marker is **empty**.

## Standards-reference accounting

The mechanical scan reported **"(no standards references found in the application tree)."** There is nothing to account for: no RFC, NIST/FIPS, ITU-T, W3C, or ISO/IEC document governs this folder. EEx is not a standardized template language (unlike, say, ERB it has no external spec); its grammar is defined by its own implementation.

## Applicable standards / contracts

### External standards
**None.** No IETF RFC, NIST/FIPS, ITU-T, W3C, ISO/IEC, or de-facto external data/report format constrains this folder. The compliance pass has nothing to adjudicate here.

### OTP-internal / Elixir-internal contracts (per-file context, NOT compliance obligations)
These are guidance for the per-file reviewer only — they are behaviour/`@doc`/`@spec` contracts, not external standards:

* **`EEx.Engine` behaviour** — callbacks `init/1`, `handle_body/1`, `handle_text/3` (and `/2`), `handle_expr/3`, `handle_begin/1`, `handle_end/1`. An engine threads an opaque state through these and returns a quoted Elixir expression from `handle_body/1`. `EEx.Engine` provides default implementations; `EEx.SmartEngine` builds on them.
* **Public API return shapes** (`EEx` module `@doc`/`@spec`): `compile_string/2` and `compile_file/2` return a Macro AST (`Macro.t()`); `eval_string/3` and `eval_file/3` return the evaluated term; `function_from_string/5` / `function_from_file/5` define a function as a side effect. Malformed template syntax is documented to raise `EEx.SyntaxError` (with `file`/`line`/`column`/`snippet`), not an arbitrary error.
* **Tokenizer error contract** — `EEx.Compiler.tokenize/2,3` returns `{:ok, ...}` / `{:error, message, meta}` internally; the compiler converts a tokenizer `{:error, ...}` into a raised `EEx.SyntaxError` at the documented boundary.
* **Deprecation contract** — deprecated syntax (`<%# ... %>`) must emit `IO.warn/2` and continue to work until the documented removal (v2.0), not silently change behaviour.

## Checkable sub-claims

These are the internal-contract obligations a per-file reviewer can realistically check against a single file in this folder. (No external-standard sub-claims exist — the verdict is `no`.)

1. **SYNTAX_ERROR_RAISES_EEX_SYNTAXERROR** — Malformed templates (unterminated tag, unexpected end, bad marker) must surface as `EEx.SyntaxError` carrying `file`/`line`/`column`, not as a raw `MatchError`/`FunctionClauseError` or a bare `throw`. (`EEx` `@doc`; `EEx.SyntaxError`.)
2. **TOKENIZE_RETURNS_TAGGED_TUPLE** — Internal `tokenize`/`comment`/expression-scanning helpers must return `{:ok, ...}` or `{:error, message, meta}` and never let a partial-match fall through to a badmatch. (Compiler internal contract.)
3. **ENGINE_STATE_THREADED** — Every `handle_*` engine callback must accept and return the engine state (or the documented buffer/quoted form) so state is never dropped between text and expr handling. (`EEx.Engine` behaviour.)
4. **HANDLE_BODY_RETURNS_QUOTED** — `handle_body/1` (and `handle_expr/3`, `handle_text/3`) must return a valid quoted Elixir expression (`Macro.t()`), not a raw string or side-effecting IO. (`EEx.Engine` behaviour / `@spec`.)
5. **LINE_COLUMN_TRACKED** — The tokenizer must advance `line`/`column` consistently across newlines, trimming, and multi-char markers so error positions and generated code line info are correct. (Diagnostic contract; historically an off-by-one source.)
6. **TRIM_PRESERVES_ONE_NEWLINE** — With `:trim` enabled, whitespace collapsing must retain exactly one newline as documented, not strip all newlines or corrupt non-whitespace content. (`EEx` `:trim` option `@doc`.)
7. **QUOTATION_TAG_EMITS_LITERAL** — The `<%% %>` quotation tag must reproduce its contents verbatim as literal text (escaping the marker), never evaluate them as code. (`EEx` tag `@doc`.)
8. **COMMENT_TAG_DISCARDED** — `<%!-- --%>` comment contents must be discarded from output and must not be evaluated or leaked into the compiled AST. (`EEx` tag `@doc`.)
9. **DEPRECATED_SYNTAX_WARNS** — Deprecated `<%#` comment syntax must call `IO.warn/2` with line/column/file and still tokenize correctly until the documented removal. (Deprecation contract.)
10. **SMART_ENGINE_ASSIGNS_CONTRACT** — `EEx.SmartEngine`'s `@` / `assigns` handling must read from the `assigns` keyword/map as documented and fail with a clear message on a missing assign rather than an opaque `KeyError` deep in generated code. (`EEx.SmartEngine` `@doc`.)
11. **UNKNOWN_OPTION_IGNORED_OR_VALIDATED** — Public entry points must apply documented defaults for absent options (`:file` → `"nofile"`, `:line` → `1`, `:indentation`/`:column` → `0`/`1`, `:engine` → `EEx.SmartEngine`, `:trim` → `false`) and not crash on option absence. (`EEx` Options `@doc`.)
12. **NO_RESOURCE_LEAK_ON_FILE_READ** — `eval_file`/`compile_file`/`function_from_file` must read the file and propagate read failures as normal errors, releasing any handle on the error path. (Secure-coding / OTP convention.)

## Per-file checking instructions

For each sub-claim that could plausibly apply to the file under review, search for the code that satisfies it. Because this folder is governed only by **internal Elixir/OTP contracts** (there are no external protocol/algorithm standards), treat every sub-claim as *per-file review guidance*, **not** as an RFC-compliance obligation — the downstream compliance pass has nothing external to adjudicate here. If a file implements a tokenizer branch, an engine callback, or a public entry point and the corresponding contract (tagged-tuple return, quoted-AST return, `EEx.SyntaxError` on bad input, correct line/column advance, documented default) is not honoured, report it as a correctness/robustness finding under the appropriate failure-mode class (badmatch/crash-on-malformed-input, dropped-state, position off-by-one, resource leak). If a file legitimately does not touch a given claim (e.g. `engine.ex` never reads files), skip that claim silently.