This is EEx (Embedded Elixir) — a template compilation engine. It compiles embedded Elixir tags into an AST/quoted expression. No network protocol, wire format, or cryptographic algorithm is implemented here; the only correctness contracts are OTP/Elixir-internal (behaviour callbacks, `-spec`/`@doc` shapes).

<!-- rfc-governed: no -->
<!-- standards: -->

# EEx (Embedded Elixir) — Standards / Contract List

## Applicable standards / contracts

### External standards
None. This folder implements a **template compilation engine** (`EEx`, `EEx.Compiler`, `EEx.Engine`, `EEx.SmartEngine`). It does not implement any IETF/ITU-T/W3C/ISO/NIST/FIPS protocol, wire format, or cryptographic algorithm. There is no on-the-wire behaviour, negotiation, state machine, or standardized algorithm whose correctness a single-file reviewer could not derive from the code itself. (Note: EEx does **not** perform HTML escaping / output encoding — that responsibility lives in downstream libraries such as `Phoenix.HTML`/`Phoenix.HTML.Safe`; EEx's `SmartEngine` only inserts a `to_string`/`String.Chars` conversion. So even the XSS/HTML-spec angle does not attach here.) The compliance pass therefore has nothing to adjudicate.

### OTP-internal contracts (per-file context, NOT compliance obligations)
These are Elixir/OTP-internal behaviour and API contracts, provided only as guidance to the per-file reviewer. The compliance pass MUST NOT treat them as standard-conformance obligations:

- **`EEx.Engine` behaviour callbacks** — `init/1`, `handle_body/1`, `handle_text/2..3`, `handle_expr/3`, `handle_begin/1`, `handle_end/1`. Custom/smart engines must return well-formed quoted expressions and thread the accumulator/buffer state consistently.
- **Public API return-shape `@spec`s** — `eval_string/3`, `eval_file/3`, `compile_string/2`, `compile_file/2`, `function_from_string/5`, `function_from_file/5` return values (`{term, binding}` for eval; Macro AST for compile).
- **`EEx.SyntaxError` exception contract** — carries `:file`, `:line`, `:column`, `:snippet` and formats via `Exception.format_file_line_column/4`.
- **Tokenizer/tag contract** — recognized tags (`<% %>`, `<%= %>`, `<%% %>`, `<%!-- --%>`, `<%|`, middle/end markers), the `:trim` and `:indentation` options, and column/line accounting for error reporting.
- **`:parser_options` passthrough** — `Code.string_to_quoted/2` options, with `:file`/`:line`/`:column` documented as ignored.

## Checkable sub-claims
None derived from an external protocol/algorithm standard — the folder is governed only by Elixir/OTP-internal contracts, which are enumerated above for the per-file reviewer's context but are out of scope for the compliance pass. (Any genuine defects here — e.g. accumulator mishandling, unescaped injection into generated AST, tokenizer position miscounts, or crashes on malformed templates — are correctness/security-review concerns, not RFC/standards-conformance gaps.)

## Per-file checking instructions
This folder is **not** RFC/standard-governed, so there are no external compliance sub-claims to check. A per-file reviewer should ignore the (empty) external-standards list and treat the OTP-internal contracts above purely as background context — e.g. verify custom engine callbacks return valid quoted expressions and thread buffer state correctly, and that tokenizer line/column accounting stays consistent — reporting any problems under the ordinary correctness/security finding classes, not as standard-conformance gaps.