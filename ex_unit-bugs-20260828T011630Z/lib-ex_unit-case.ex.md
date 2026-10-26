# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/case.ex
<!-- sha256: 9cdb0ec478d8b3830e2f6b7e17a0d40085ddd161fb61e3982fb02adfb865f8b9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:28:04Z_

# Bug-Hunting Review — `elixir/lib/ex_unit/lib/ex_unit/case.ex`

## Summary
After a full read and runtime verification of the non-obvious paths (long/multibyte test-name truncation, tag normalization/precedence, reserved-tag and parameterize validation, and helper-function existence), I found no defect reachable through this module's documented public API.

## Bugfixes

No findings.

The paths most likely to harbor a defect were exercised directly and behaved correctly:

- **`build_test_name/1` (>255-byte recursion, case.ex:922-938).** I feared `String.byte_slice(name, 0, 246)` could split a multibyte UTF-8 codepoint, producing an invalid binary fed to `String.to_unsafe_atom/1`. Verified `String.byte_slice/3` truncates to the codepoint boundary (asking for 245 bytes of 2-byte chars returns 244; 246 of 3-byte chars returns 246, both `String.valid?` true). End-to-end, a 360-byte multibyte test name compiles to a valid atom of 253 bytes (`atom name valid utf8: true`). The truncation budget (246 + `"... "` (4) + 4-char base64 hash = 254 ≤ 255) is correct and the recursion terminates in one step.
- **Tag precedence in `normalize_tags/1` + `register_test/6` (case.ex:714-725, 940-956).** The combine-then-`Enum.reverse` ordering yields the documented precedence `@tag` > `@describetag` > `@moduletag`, and "last value wins" for a repeated key, both within a single keyword `@tag` and across lines.
- **`validate_tags/1` reserved-tag guard (case.ex:910-920).** `@tag :async` correctly raises `cannot set tag :async because it is reserved by ExUnit`; `tags[:test_type]` being `nil` for module tags passes the `is_atom/1` check as intended.
- **`__before_compile__/1` parameterize/async validation (case.ex:616-622).** `parameterize: [1, 2]` correctly raises `:parameterize must be a list of maps`.
- **`get_last_registered_test/1` (case.ex:906-908).** `Module.get_last_attribute/2` exists and is exported (an initial `function_exported?` false-negative was due to `Module` not yet being loaded in that eval), and it raises on an already-compiled module, matching the `@doc` "Raises if called with a module that has already been compiled."

## Risks / caveats
- The `describe/2` macro (case.ex:574-593) delegates most of its behavior to `ExUnit.Callbacks.__describe__/3,4`, which lives in a different file and was out of scope; I confirmed only that the local AST wiring (`quote unquote: false` filling `{name, body}` into the `defp` definition) is well-formed, not the callback's own logic.
- `annotate_test/1` (case.ex:481) gates on the internal `:ex_unit`/`:breakpoints` env with `and` (strict boolean); a non-boolean misconfiguration would raise `BadBooleanError`, but that env is not part of the documented public API, so it is out of scope.

### Ruled out
- **`test/3` non-tuple context warning (case.ex:432-438).** The `is_tuple(var)` heuristic is correct: every valid pattern AST (variables, `%{}`, `%Struct{}`, `_`, `=`-patterns) is a tuple, so it never false-warns; the default `quote(do: _)` is a tuple and suppresses the warning. Non-tuple literals (a bare list/atom pattern) genuinely can never match the map context, so the warning is accurate.
- **Doc lists more "reserved" tags than `@reserved` guards (case.ex:151-187 vs 351).** Verified `@tag test_pid: :bogus` is accepted (not guarded), whereas the "Known tags" doc calls `:test_pid` reserved. This is a doc-emphasis mismatch — those tags are set/overwritten automatically at runtime by the Runner, so a user value is silently overridden rather than causing incorrect behavior. Too minor and non-defective to report as a bugfix.
- **`register_attribute` duplicate/ordering logic (case.ex:866-887).** `validate_registered_attribute!` correctly rejects cross-type re-registration and setting `@name` before registration; accumulate ordering (newest-first) matches the `@doc` examples.
- **`test/1` "not implemented" macro (case.ex:503-510).** `bind_quoted: binding()` captures `message`, `mod`, `file`, `line`; the generated `def name(_), do: flunk(...)` relies on `flunk/1` imported via `use ExUnit.Case` — present and correct.

