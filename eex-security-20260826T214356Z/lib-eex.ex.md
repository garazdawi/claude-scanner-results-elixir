# /Users/lukas/code/claude-scanner/elixir/lib/eex/lib/eex.ex
<!-- sha256: 807aea9052f92eedc2747b586257bb93d563d719e69328050bc8ac317d4d68d5 -->

_Analysed on 2026-08-26T21:50:29Z_

I have enough to conclude. Let me record what I verified and finalize.

**Verified facts from running the built toolchain:**
- `eval_string`/`eval_file` execute the template as code (SSTI by design, honestly named — `eval`).
- `compile_string` and `tokenize` do **not** execute template side effects (execution is deferred to `Code.eval_quoted` in `do_eval`); so compiling/tokenizing an untrusted template is side-effect-free.
- `bindings` is a pure data channel — values cannot become code.
- `EEx.SyntaxError.message/1` (in this file) does pass attacker-influenceable raw template bytes (ESC/BEL survived into `Exception.message/1`), but that snippet/message is constructed in `EEx.Compiler`, this file has no terminal/log sink, and the function is idiomatic exception behavior.

## Summary

`eex.ex` is a thin, honestly-named public wrapper over `EEx.Compiler`; its only powerful sinks — template compilation and `Code.eval_quoted` — are the documented purpose (template *is* code), reachable only when the caller itself chooses to eval an untrusted template, so there are no security defects attributable to this file.

## Security

No findings.

### Ruled out

- **SSTI → RCE via `eval_string/3`, `eval_file/3`, `function_from_*` (template is code).** Verified `EEx.eval_string("2+2=<%= 6*7 %>") => "2+2=42"`. This is the documented contract (the module doc, tags `<% %>`/`<%= %>`, and the `eval`/`compile` names all say "template is Elixir code"). Per scope, a victim choosing to `eval`/`compile` attacker input is not a defect of this repository. `bindings`/`@assigns` is the safe data channel — verified a binding value lands as inert data, not code.
- **`EEx.SyntaxError.message/1` echoes raw template bytes (CWE-117 / terminal-escape injection).** Verified: a tokenizer-stage error carries the source line verbatim into `Exception.message(e)` — raw `\e[31m` and `\a` (bytes `27,91,51,49,109,7`) survived. However: (1) this file has *no* terminal/log/`~s` sink — `message/1` only returns a string; the injection only materializes if a downstream consumer writes it unsanitized, which is the consumer's boundary; (2) the raw snippet/message content is built in `EEx.Compiler`, not here; (3) returning the raw diagnostic is idiomatic for every Elixir/OTP exception `message/1`, and sanitizing here would corrupt legitimate error output. Latent defense-in-depth at most, and not localized to this file.
- **Compile-time file read in `function_from_file/5` (`EEx.compile_file` → `File.read!`) and `@external_resource`/`@file`.** Runs on the build machine, but the path is a literal in the caller's own source — operator-controlled, by design.
- **`compile_file/2` / `eval_file/2` `File.read!(filename)` — path traversal / unbounded read.** The caller names the file; there is no root to confine to and no external path component. Documented behavior.
- **`:engine` and `:parser_options` passthrough.** `:engine` selects a module whose callbacks run at compile time; `:parser_options` is forwarded to `Code.string_to_quoted!` (e.g. `:static_atoms_encoder`). Both are operator-supplied options in the API contract, not attacker-reachable through this file.
- **`do_eval/3` options (`:file`,`:line`,`:module`,`:prune_binding`) to `Code.eval_quoted`.** Internal `Keyword.take` of caller options; no code-execution escalation beyond the already-documented eval.
- **Atom creation.** None in this file; any `String.to_atom`/`list_to_atom` lives in `EEx.Compiler`/`EEx.Engine`, out of this file's scope.

## Risks / caveats

- This file is a wrapper: the substantive tokenizer/compiler logic, atom creation, snippet/index math, and any parser/resource-DoS surface live in `EEx.Compiler` and `EEx.Engine` (separate files, per the folder threat model's cross-file invariants). Nothing here validates that template *content* is safe — by design, every entry point assumes the caller established template trust.
- The single most important operational caveat for consumers of this API: **the template argument is code, not data.** `eval_string`/`eval_file`/`function_from_*`/`compile_string` on any template that crosses a trust boundary (user-submitted, uploaded `.eex`, DB-stored) is arbitrary RCE — EEx does no sandboxing. Only `bindings`/`@assigns` is a safe channel for peer data. This is inherent to the engine, not a fixable defect in this file.
- EEx performs **no HTML/output escaping** (`<%= %>` runs `String.Chars.to_string/1`); XSS defense is the downstream engine's job (e.g. `Phoenix.HTML`). Not a defect of this folder.
- I could not fully exercise the `EEx.SyntaxError` struct with a caller-supplied non-binary `:file` (a contract violation, out of scope); `file && Path.relative_to_cwd(file)` assumes `file` is `nil` or a binary per the `@type`/docs.

