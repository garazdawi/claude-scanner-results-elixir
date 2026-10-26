# /Users/lukas/code/claude-scanner/elixir/lib/eex/lib/eex/compiler.ex
<!-- sha256: 01b4c3e489fb47b17d7670bbc0178e3169f6d14511a2e9cf912a62c88d67c37e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:09:57Z_

## Summary
Two defects in `EEx.Compiler`: a documented-valid `column: 0` turns any template syntax error into an unhelpful `ArgumentError` instead of `EEx.SyntaxError`, and the deprecated `<%#` comment path drops the code snippet from its error message.

## Bugfixes

#### `CSCAN-compiler-code-snippet-column-zero`
- **Finding ID** — `CSCAN-compiler-code-snippet-column-zero`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `compiler.ex:570`, `code_snippet/3` (anonymous `map_reduce` clause)
- **Class** — Contract violation (crash where the contract promises `EEx.SyntaxError`)
- **Issue** — The error-snippet arrow is built with `String.duplicate(" ", meta.column - 1)`. When `meta.column` is `0`, the count is `-1` and `:binary.copy(" ", -1)` raises `ArgumentError`. `meta.column` is `0` whenever the caller passes `column: 0`, which is a valid value: `EEx`'s published type is `@type column :: non_neg_integer` (eex.ex:112), so `0` conforms to the `@spec`, not violates it. Any malformed template then raises `ArgumentError` instead of the documented `EEx.SyntaxError`.
- **Impact** — A caller using 0-based column indexing gets, for every template syntax error, a crash from deep inside `:binary.copy/2` that names nothing about the template problem — no file, line, or "unterminated tag" message. Reachable through `EEx.tokenize/2`, `EEx.compile_string/2`, `EEx.compile_file/2`, `EEx.eval_string/3`, etc., all of which forward `:column`. Template source is a documented public entry point.
- **Evidence** — `verified`.\
  `elixir -e 'EEx.tokenize("<% bad", column: 0)'` →
  ```
  ** (ArgumentError) errors were found at the given arguments:
    * 2nd argument: out of range
      (stdlib) :binary.copy(" ", -1)
      lib/eex/compiler.ex:570: anonymous fn/7 in EEx.Compiler.code_snippet/3
  ```
  With `column: 1` the same input yields a clean `** (EEx.SyntaxError) nofile:1:1: expected closing '%>' for EEx expression` with a snippet.
- **Suggested fix** — Clamp the width: `String.duplicate(" ", max(meta.column - 1, 0))`.

#### `CSCAN-compiler-deprecated-comment-no-snippet`
- **Finding ID** — `CSCAN-compiler-deprecated-comment-no-snippet`
- **Severity** — Low
- **Confidence** — high
- **Location** — `compiler.ex:63-65`, `tokenize/6` `<%#` clause
- **Class** — Documentation mismatch (contract inconsistency)
- **Issue** — On an unterminated deprecated `<%#` comment, the error is returned as `{:error, message, %{line: line, column: column}}` with no `code_snippet/3` appended, unlike the `<%` (line 79) and `<%!--` (line 47) paths which both append the snippet. The resulting `EEx.SyntaxError` therefore carries no `snippet`.
- **Impact** — Cosmetic but real: an error in the still-supported `<%#` syntax produces a worse diagnostic (no source excerpt / caret) than every other tag, inconsistent with the SyntaxError-carries-snippet contract.
- **Evidence** — `verified`. `EEx.compile_string("foo\nbar <%# comment no close")` raises `EEx.SyntaxError` whose message is bare (`nofile:2:5: expected closing '%>' for EEx expression`), whereas the identical `<%` input prints the `1 | foo` / `2 | bar ... ^` snippet block.
- **Suggested fix** — Mirror the `<%` clause: `{:error, message <> code_snippet(state.source, state.indentation, meta), meta}` with `meta = %{line: line, column: column}`.

### Ruled out
- **`:after` missing from `clause_block_identifier?`** (line 503) — suspected to break `try/after` template merging, but `verified` that `<%= try do %>a<% after %>b<% end %>` and `rescue`+`after` compile and evaluate correctly; the whitespace-merge only fires between two `middle_expr` tokens, which `after` clauses do not produce.
- **`maybe_append_space/1` has no `[]` clause** (line 235) — `verified` unreachable: it is only called after `token_key` matches a `{:do,_}`/`block_identifier` token sequence, which requires a non-empty expression charlist.
- **`parser_options ++ nil` improper list** (line 330) — `Code.get_compiler_option(:parser_options)` returns `[columns: true]` by default, so the concatenation is a proper list; producing `nil` requires deliberately corrupting a global compiler option (misuse).
- **Trim retaining/removing newlines** — trim output was inspected against the `:trim` doc; behavior matches the long-standing implementation and no concrete deviation from "one newline retained" could be demonstrated on valid input.

## Risks / caveats
- I could not diff against an upstream reference; trim arithmetic (`trim_left`/`trim_right`/`trim_init`) and the `line`/`column` threading are the fiddliest parts of the file, and a subtle off-by-one in generated-code line numbers (silent, not a crash) would not surface through the black-box tests I ran.
- The `token_key` block-classification path (`closing_bracket?`, `fn_before_end?`, `drop_eol`) was exercised only through common block forms; exotic Elixir constructs embedded in tags were not exhaustively fuzzed.
- Findings assume the installed `/opt/elixir` (1.21.0-dev) is built from the source under review, which the line numbers in the reproduced stack traces confirm.

