# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/doc_test.ex
<!-- sha256: f9d76dde1c62df9a8722503e5fce58cf32971960bb12f075fbce39aa4d7766a6 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/doc_test.ex
<!-- sha256: f9d76dde1c62df9a8722503e5fce58cf32971960bb12f075fbce39aa4d7766a6 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

## Summary
Re-ran all three prior defects against the source under review (sandbox: Erlang/OTP 29, Elixir 1.21.0-dev) — every one reproduces, so all three are preserved with unchanged severities. The two first-pass hedges ("did not exhaustively fuzz the `adjust_indent`/`adjust_code` state machine or the `error_message_matches?` ellipsis handling") were resolved by execution rather than assumption: `error_message_matches?` was pushed through every ellipsis axis (multibyte, bare, empty-actual, short-actual, mid-string) and is clean; `adjust_code` line attribution and overrun detection were exercised and are correct — which independently corroborates that finding 1's off-by-one lives in the `__doctest_file__` base value, not in the shared line machinery. No hedge remains as an unverified assumption.

## Bugfixes

#### `CSCAN-doc_test-doctest-file-line-off-by-one`
- **Finding ID:** `CSCAN-doc_test-doctest-file-line-off-by-one`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `doc_test.ex:303` (`__doctest_file__/3`) together with `doc_test.ex:831` (`extract_tests/4`, the `adjust_indent(line_no + 1, module)`).
- **Class:** Off-by-one / wrong arithmetic
- **Impact:** Every doctest extracted from a markdown file via the documented `doctest_file/2` macro is attributed to a line **one greater** than its real position. A developer clicking the failure stacktrace lands on the wrong line of the README/markdown file. Consistent for all `doctest_file` users (a public macro since 1.15.0).
- **Evidence:** `verified` (re-run this pass). A file whose only content is `    iex> 2 + 2` (file line 1) and `    5` (line 2) reports the failure at `R2.md:2`:
  ```
       iex> 2 + 2
       R2.md:2: R2Test (module)
  ```
  Root cause confirmed against the shared machinery: `adjust_code`/`extract_tests` attribute lines correctly for `@doc`/`@moduledoc` sources (see the overrun probe under Ruled-out, which lands on the exact right line). The discrepancy is solely that `__doctest_file__` passes a base of `1` for a file whose content *begins* on line 1, so `extract_tests`' internal `line_no + 1` double-counts.
- **Suggested fix:** Call `extract_tests(0, doc, module, :moduledoc)` in `__doctest_file__/3` (so the internal `line_no + 1` yields `1` for the first line), or thread a flag so `extract_tests` skips the `+1` for file sources.

#### `CSCAN-doc_test-expected-line-nested-iodata-length`
- **Finding ID:** `CSCAN-doc_test-expected-line-nested-iodata-length`
- **Severity:** Low
- **Confidence:** high
- **Location:** `doc_test.ex:905` (`add_expr/5`): `expected_line: line_no + length(expr_lines)`.
- **Class:** Off-by-one / wrong arithmetic (nested-iodata length)
- **Impact:** For a doctest expression spanning **3 or more source lines** (an `iex>` line plus two or more `...>` continuations) that has an expected result, the line number attributed to the *expected* expression is wrong — off by `(k − 2)` where `k` is the number of expression lines. This misplaces the reported line when the expected expression fails to compile or raises at runtime.
- **Evidence:** `verified` (re-run this pass). `expr_lines` is built by `add_line`, which produces *nested* iodata, so `length/1` counts top-level cells, not lines — reproduced directly:
  ```
  1 lines -> length=1
  2 lines -> length=2
  3 lines -> length=2
  4 lines -> length=2
  ```
  A 4-line expression (`iex> a = 1` … `...> a + b + c`) with an un-compilable expected `6 +` on file line 7 reports `mod.ex:5` / `line: 5` (should be 7) — off by 2:
  ```
  Doctest did not compile ... token missing on mod.ex:5:3
  [{MyMod2, :__MODULE__, 0, line: 5, file: ~c"mod.ex"}]
  ```
- **Suggested fix:** Track the true line count separately (e.g. carry an accumulator counter through `build_test`, or `expected_line: line_no + Enum.count(String.split(IO.iodata_to_binary(expr_lines), "\n"))`) instead of `length(expr_lines)`.

#### `CSCAN-doc_test-malformed-exception-matcherror`
- **Finding ID:** `CSCAN-doc_test-malformed-exception-matcherror`
- **Severity:** Low
- **Confidence:** high
- **Location:** `doc_test.ex:918` (`tag_expected/1`): `[mod, message] = :binary.split(error, ")")`.
- **Class:** Missing clause / non-total function (pattern match on wrong shape)
- **Impact:** A doctest whose expected line begins with `** (` but is missing the closing `)` (a realistic authoring typo) crashes compilation with a raw `MatchError` carrying no file/line and no hint at the offending doctest — bypassing the `ExUnit.DocTest.Error` mechanism that exists precisely to give an author-friendly message with source location.
- **Evidence:** `verified` (re-run this pass, compiling the module to a BEAM with docs so `Code.fetch_docs` succeeds). A doctest with expected `** (RuntimeError` (no closing paren):
  ```
  ** (MatchError) no match of right hand side value:
      (ex_unit 1.21.0-dev) lib/ex_unit/doc_test.ex:918: ExUnit.DocTest.tag_expected/1
      (ex_unit 1.21.0-dev) lib/ex_unit/doc_test.ex:904: ExUnit.DocTest.add_expr/5
      (ex_unit 1.21.0-dev) lib/ex_unit/doc_test.ex:839: ExUnit.DocTest.build_test/2
  ```
- **Suggested fix:** Match the split defensively, e.g. `case :binary.split(error, ")") do [mod, message] -> ...; [_] -> raise Error, module: module, message: "..." end`, raising a `DocTest.Error` explaining the exception line must be `** (Exception) message`.

## Risks / caveats
- All three findings are diagnostic-quality defects (wrong line attribution, unhelpful crash message) rather than incorrect pass/fail verdicts — I found no path where a doctest passes or fails *incorrectly* on valid input. Impact is developer time and error legibility.
- The first-pass caveat about `error_message_matches?` and the indentation state machine has been discharged by execution (see Ruled out); it is no longer an open assumption.

### Ruled out

- **`error_message_matches?` ellipsis (`binary_slice(expected, 0..-4//1)`, `doc_test.ex:608–617`)** — no defect.
  - Axes examined: multibyte bytes preceding the ellipsis (`"café ..."`), ellipsis boundary landing mid-string, bare `"..."` expected, empty-actual vs bare ellipsis, actual shorter than the stripped prefix, and a two-dot `".."` non-terminator. Executed a faithful replica of lines 608–617 and observed: `"some error in pid: ..."` → matches `"...: <0.1>"`; `"café ..."` → true; `"..."` matches both `"anything"` and `""` (stripped prefix becomes `""`, `starts_with?` is universally true — the documented multiline-message behaviour); `"abcd..."` vs actual `"ab"` → false (correctly rejects); `".."` → false (not `ends_with?("...")`). Byte-slicing of the last 3 bytes only ever removes the three ASCII dots, so multibyte content earlier in the string is untouched.
  - Not examined: none.

- **`adjust_indent` / `adjust_code` line attribution and overrun detection (`doc_test.ex:711–786`)** — no defect found on the axes tried.
  - Axes examined: a code/result line indented deeper than its `iex>` (`:code`-kind overrun) → correctly raises `ExUnit.DocTest.Error` at the exact offending file line (`mod.ex:12`) with the "indentation level mismatch … exactly 4 spaces" message; blank-line and fenced-block terminators between two examples parse without bleeding lines across examples; multiline `iex>`/`...>` continuation chains (the 4-line expression in finding 2) walk the state machine and land at the correct `iex>` base line. Line attribution through this machinery is accurate — the finding-1 off-by-one is confined to the `__doctest_file__` base argument, not this code.
  - Not examined: tab-indented prompts (byte-offset `strip_indent` under mixed tabs/spaces), the `~~~` fence variant specifically (only ` ``` ` exercised), and CRLF-split docs. These stay on the not-examined list; this entry remains `asserted` for those axes.

- **`skip_iex_number`/`adjust_prompt` on numbered prompts** — traced `iex(1)>`, `...(1)>`, and empty `iex()>`; all split correctly on `")>"`; the no-match branch raises a clear `DocTest.Error` with line and accepted formats. Behaves as documented.
  - Axes examined: numbered, empty-numbered, and dot-continuation numbered prompts; missing-`)>` no-match branch. Not examined: none of consequence for the shape checked.

- **`filter_by_opts`/`filter_tests` `:only`/`:except`** — `{:ok, []}` short-circuits to `[]`; undefined `{fun, arity}` entries raise a formatted `DocTest.Error`; `Stream.reject` composes with the downstream `Enum.sort_by`. No missing-clause or silent-empty bug found.

- **`multiple_exceptions?` / `test_content` guard** — correctly raises a `DocTest.Error` for >1 exception expectation in one example, matching the documented "separate your iex> prompts" guidance.

### Did I verify this
- **Finding 1 (doctest_file off-by-one):** verified — re-reproduced end-to-end via the documented `doctest_file/2` macro; `iex>` on file line 1 reported at `R2.md:2`.
- **Finding 2 (nested-iodata length):** verified — re-reproduced the off-by-2 report through `doctest/1`, and independently confirmed the root cause by executing the `add_line`/`length` construction (`length` saturates at 2 for ≥2 lines).
- **Finding 3 (malformed exception MatchError):** verified — re-reproduced the raw `MatchError` at `doc_test.ex:918` by compiling a docs-bearing BEAM and running `doctest/1` over a `** (RuntimeError` expectation missing its `)`.
- **`error_message_matches?` ellipsis:** verified clean — executed a replica over all listed axes; every axis tried, none remaining.
- **`adjust_code`/`adjust_indent`:** partially verified — overrun/line-attribution/fence/multiline axes executed and clean; tab/`~~~`/CRLF axes explicitly left on the not-examined list (entry stays `asserted` for those).

### Dropped in re-review
None. All three prior findings re-reproduced this pass and are preserved unchanged. No prior finding was upgraded from a hedged assumption (the first-pass hedges were caveats on already-ruled-out areas, now discharged by execution rather than converted to findings).
