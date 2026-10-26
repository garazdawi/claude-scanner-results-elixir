# Cross-file findings

## Asserted-dependency ledger

No per-file report used the literal `* Depends on:` sentinel; the dependencies below were extracted from the prose "depends on" / "out of scope, lives in X" / "must be reviewed separately" hedges and the threat-model cross-file invariants.

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| lib-eex.ex.md (finding #2, assign doc) | `engine.ex` `fetch_assign!/2` warn-and-`nil` on missing assign | Confirmed | engine.ex Ruled-out "Missing-assign path in `fetch_assign!/2`" is `verified` (warns + renders `""`). |
| lib-eex.ex.md (finding #2, assign doc) | `smart_engine.ex` `handle_expr/3` prewalk rewrites `@foo` via `EEx.Engine.handle_assign/1` | Confirmed | smart_engine.ex Ruled-out "`handle_expr/3` prewalk" `verified` it forwards to `EEx.Engine.handle_assign`. |
| lib-eex.ex.md (finding #1, `eval_file` spec) | `engine.ex` `handle_body/1 :: Macro.t()` may return non-binary | Confirmed | engine.ex Ruled-out "State threading / `handle_body`" confirms `HANDLE_BODY_RETURNS_QUOTED` (arbitrary quoted term). |
| lib-eex.ex.md (Risks, line 50) | `compiler.ex` honours `SYNTAX_ERROR_RAISES_EEX_SYNTAXERROR` (template error → `EEx.SyntaxError`) | **Contradicted** | compiler.ex `CSCAN-compiler-code-snippet-column-zero`: valid `column: 0` → `ArgumentError`, not `EEx.SyntaxError`. |
| lib-eex-smart_engine.ex.md (Risks) | `engine.ex` adjudicates `SMART_ENGINE_ASSIGNS_CONTRACT` (missing-assign warn-vs-raise) | Confirmed | engine.ex `verified` the warn-and-`nil` path; contract satisfied. |
| lib-eex-engine.ex.md (Risks) | `compiler.ex` (tokenizer/compiler boundary feeding callbacks) reviewed separately | Confirmed | compiler.ex has its own report; no contradiction of engine.ex's callback assumptions. |
| lib-eex-engine.ex.md (Risks) | Callers/engines always pass well-formed `%{binary,dynamic,vars_count}` state to the `handle_expr ""` clause (no `check_state!`) | Unresolved | No report verifies the compiler always supplies valid state to the un-checked `""` clause; internal-misuse framing untested. |
| lib-eex-compiler.ex.md (Risks, line 51) | Installed `/opt/elixir` 1.21.0-dev is built from the source under review | Confirmed | mix.exs.md `verified` `System.version()` → `1.21.0-dev`; stack-trace line numbers match. |
| THREAT_MODEL.md (`handle_text/2` vs `/3` invariant) | `engine.ex` deprecated `__using__` defining only `handle_text/2` still works via `function_exported?` fallback | Confirmed | engine.ex Ruled-out "Deprecated `__using__` … only `handle_text/2`" `verified` correct output. |

## Findings

#### `CSCAN-crossfile-column-zero-breaks-syntaxerror-contract`
- **Finding ID** — `CSCAN-crossfile-column-zero-breaks-syntaxerror-contract`
- **Severity** — Medium
- **Confidence** — high
- **Files involved** — `lib-eex.ex.md`, `lib-eex-compiler.ex.md`, `THREAT_MODEL.md`
- **Issue** — `lib-eex.ex.md` (Risks/caveats, line 50) asserts that `compile_string/2` "correctly converts a `{:error, message, meta}` tuple into a raised `EEx.SyntaxError` … honouring the `SYNTAX_ERROR_RAISES_EEX_SYNTAXERROR` contract at this boundary" (`eex.ex:245-247`), treating the compiler as a trusted neighbour that always surfaces template errors as `EEx.SyntaxError`. `lib-eex-compiler.ex.md` finding `CSCAN-compiler-code-snippet-column-zero` (`compiler.ex:570`) verifies the opposite for a documented-valid input: with `column: 0` (which conforms to `@type column :: non_neg_integer`, `eex.ex:112`), `code_snippet/3` computes `String.duplicate(" ", -1)` and raises `ArgumentError` *before* any `{:error, …}` tuple is produced — so eex.ex's conversion code is never reached and the public entry points that forward `:column` (`EEx.compile_string/2`, `eval_string/3`, `tokenize/2`) violate the very `SYNTAX_ERROR_RAISES_EEX_SYNTAXERROR` invariant the façade claims to uphold, and the threat model's "dominant failure is raise-as-`EEx.SyntaxError`, not an opaque crash" expectation. The per-file eex.ex review could not see this because it scoped the syntax-error contract to its own tuple-conversion code and deferred the upstream compiler logic as "not under review here."
- **Resolution** — Apply the compiler.ex fix (`max(meta.column - 1, 0)` at `compiler.ex:570`); then eex.ex's boundary claim holds for the full documented `column` domain. Until then, upgrade eex.ex's line-50 caveat to note that the `SYNTAX_ERROR` contract it relies on is broken for `column: 0` across all `:column`-forwarding entry points.
