# EEx Bugfix Scan — Ranked Summary

Scan of `elixir/lib/eex` (tree `3389f8405`, Elixir 1.21.0-dev). No `VALIDATION.md` present; `CLUSTER_lib-eex.md` reports no cluster findings. Every finding below was reproduced by execution against the installed toolchain. The dominant issue is a single compiler bug whose blast radius the cross-file review widened to a broken public contract.

---

### 1. `column: 0` turns any template syntax error into an opaque `ArgumentError` (Medium) — cross-file, verified
**`compiler.ex:570` + all `:column`-forwarding public entry points.** In `code_snippet/3`, the caret arrow is built with `String.duplicate(" ", meta.column - 1)`. When `column` is `0` — a *documented-valid* value (`@type column :: non_neg_integer`, `eex.ex:112`) — the count is `-1` and `:binary.copy(" ", -1)` raises `ArgumentError` **before** the `{:error, …}` tuple is ever produced. So a 0-based caller gets, for *every* malformed template, a crash from deep inside `:binary.copy/2` naming nothing about the template — no file, line, or "unterminated tag".

The **cross-file finding** ([`CSCAN-crossfile-column-zero-breaks-syntaxerror-contract`](./CROSS_FILE.md#cscan-crossfile-column-zero-breaks-syntaxerror-contract)) shows the reach: `eex.ex` claims to uphold the `SYNTAX_ERROR_RAISES_EEX_SYNTAXERROR` contract at its tuple-conversion boundary (`eex.ex:245-247`), but that code is never reached — the crash happens upstream. The invariant is therefore broken across `EEx.tokenize/2`, `EEx.compile_string/2`, `EEx.compile_file/2`, `EEx.eval_string/3`, etc., all of which forward `:column`. The per-file `eex.ex` reviewer scoped this to its own code and could not see it; this is the CROSS_FILE ledger's one **Contradicted** dependency.

**Fix:** clamp the width — `String.duplicate(" ", max(meta.column - 1, 0))` at `compiler.ex:570`. This single fix also restores the `eex.ex` boundary claim across the full documented `column` domain.

---

### 2. Deprecated `<%# … %>` comment error drops its code snippet (Low) — verified
**`compiler.ex:63-65`.** On an unterminated `<%#` comment, the error is returned as `{:error, message, meta}` with no `code_snippet/3` appended — unlike the `<%` (line 79) and `<%!--` (line 47) paths. The resulting `EEx.SyntaxError` carries no source excerpt or caret, giving the still-supported `<%#` syntax a strictly worse diagnostic than every other tag. Cosmetic but a real contract inconsistency. **Fix:** mirror the `<%` clause and append the snippet.

---

### 3. `eval_file/3` `@spec` is too narrow (Low) — verified
**`eex.ex:318`.** `eval_file/3` is spec'd `:: String.t()` while its sibling `eval_string/3` is spec'd `:: term()`, though both funnel through `do_eval/3`. With a custom `:engine`, `handle_body/1` may return any `Macro.t()` (verified: a custom engine returns `42` through `eval_file/3`), so the spec is a false narrowing that mis-informs Dialyzer and downstream analysis. No runtime failure. **Fix:** change the return type to `term()` to match `eval_string/3`.

---

### 4. Moduledoc misdescribes `SmartEngine` assign resolution (Low) — verified
**`eex.ex:103-105`.** The moduledoc says `<%= @foo %>` "translates to" `<%= {:ok, v} = Access.fetch(assigns, :foo); v %>`, which would raise `MatchError` on a missing assign. The real `SmartEngine` path rewrites to `EEx.Engine.fetch_assign!/2`, which on a miss **emits `IO.warn` and returns `nil`** (verified: renders `""`, not a raise; `# TODO: Raise on v2.0` at `engine.ex:138` confirms the raise is intentionally deferred). A reader trusting the doc will mis-reason about templates that silently render `nil`. **Fix:** update the illustrative translation to the warn-and-`nil` semantics.

---

## Files with no surviving findings
- **`engine.ex`** — no findings; the missing-assign warn-and-`nil` path, the `<%/`/`<%|` marker raise, state threading, and the deprecated `__using__` wart were all verified benign. (The `handle_expr ""` clause skipping `check_state!` is flagged **Unresolved** in the CROSS_FILE ledger, but only reachable via internal engine misuse, out of scope.)
- **`smart_engine.ex`** — no findings; thin delegates plus one verified `@assign`-prewalk wrapper.
- **`mix.exs`** — no findings; static three-key project metadata with no input-carrying sink.

## Bottom line
One fix matters most: clamping `meta.column - 1` at `compiler.ex:570` eliminates the Medium crash **and** repairs the cross-file `EEx.SyntaxError` contract in one change. The remaining three are Low-severity spec/doc/diagnostic-quality issues with no runtime impact on valid default-engine input.
