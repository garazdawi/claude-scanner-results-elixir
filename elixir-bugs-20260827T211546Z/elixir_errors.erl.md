# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_errors.erl
<!-- sha256: e77a5ff61923393aed228beae7e5fe928b188b2962692c62fd5d115b67da3de7 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_errors.erl
<!-- sha256: e77a5ff61923393aed228beae7e5fe928b188b2962692c62fd5d115b67da3de7 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

# Bug-hunting review (re-review) — `elixir_errors.erl`

## Spec-totality checklist

- **`compile_error/3`** — spec Message `binary() | unicode:charlist()`. Covered: `is_binary(Message)` (310) and `is_list(Message)` (313); `unicode:charlist()` satisfies `is_list`. Total. ✓
- **`erl_warn/3`** — spec File `unicode:chardata()`. Only non-`none` clause is guarded `when is_binary(File)` (248); a list `chardata` File has no clause → `function_clause`. Reported below.
- **`parse_error/5`** — spec File `binary() | {binary(), binary()}`. The `{binary(),binary()}` File alternative is never handled; every clause forwards File to `raise_snippet/5`, guarded `when is_binary(File)` (424) → `function_clause`. Reported below.

## Summary
Mature error-formatting module. Re-review executed the four axes the first pass had only asserted. Two of them (**silent corruption**, **encoding mishandling**) are now discharged *by execution*: malformed UTF-8 does not silently corrupt — it raises `badarg`. That crash, plus the two previously-only-reasoned crash paths (`cut_snippet` `nthtail` overrun; `format_snippet/6` exotic Position), are promoted to three low-confidence crash findings, since the *mechanism* is now executed-confirmed even though the *trigger reachability* remains un-runnable here. The three original findings stand.

## Bugfixes

#### `CSCAN-elixir_errors-erl-warn-chardata-file`
- **Finding ID:** CSCAN-elixir_errors-erl-warn-chardata-file
- **Severity:** Low
- **Confidence:** low
- **Location:** `elixir_errors.erl:245-249`, `erl_warn/3`
- **Class:** Contract violation (spec accepts more than the code)
- **User-visible outcome:** A caller obeying the `-spec` and passing a `unicode:chardata()` File as a charlist gets a raw `function_clause` crash instead of a warning.
- **Reproduction:** `elixir_errors:erl_warn(none, "somefile.ex", <<"warn">>)` → first clause recurses to `erl_warn(0, "somefile.ex", ...)`, which fails `is_binary(File)`. In-tree callers pass binary File (`elixir.erl:484` `emit_warnings(Warnings, File, Opts)` where File is the `string_to_quoted` File; `elixir_erl_compiler.erl:168` `erl_warn(Line, File, Message)` with a compile-file binary), so not hit today — but `elixir.erl`'s File originates from the public `Code.string_to_quoted` file option, which a user could pass as a charlist.
- **Suggested fix:** Widen handling to list chardata (`unicode:characters_to_binary(File)`) or tighten the `-spec` File to `binary()`.
- **Risk of fix:** none (widening) / spec-only (tightening).
- **Evidence:** read `elixir_errors.erl:245-249`; `elixir.erl:481-484`, `elixir_erl_compiler.erl:166-168` (both currently binary).

#### `CSCAN-elixir_errors-parse-error-tuple-file`
- **Finding ID:** CSCAN-elixir_errors-parse-error-tuple-file
- **Severity:** Low
- **Confidence:** low
- **Location:** `elixir_errors.erl:319-320` (spec) / `elixir_errors.erl:424` (`raise_snippet/5`)
- **Class:** Contract violation / missing clause
- **User-visible outcome:** A caller passing the spec-permitted `{binary(), binary()}` File form gets a `function_clause` in `raise_snippet/5` rather than a formatted `SyntaxError`.
- **Reproduction:** `parse_error(Loc, {<<"a">>,<<"b">>}, Error, Token, Input)` reaches `raise_snippet(_, {..}, ..)` whose sole clause requires `is_binary(File)`. The one in-tree caller (`elixir.erl:525`) forwards the user's `File`. Note the *tuple* actually produced by the parser (`elixir.erl:470-471`, `{to_binary(ErrorPrefix), to_binary(ErrorSuffix)}`) is the **Error** argument (handled at clause 385), not File — so the tuple-File branch is currently unexercised.
- **Suggested fix:** Drop `{binary(),binary()}` from the File type (line 319) or add tuple handling in `raise_snippet/5`.
- **Risk of fix:** none (spec-only change) / small (adding a clause).
- **Evidence:** read `elixir_errors.erl:319-320,424-426`; `elixir.erl:513-525,470-471`.

#### `CSCAN-elixir_errors-read-snippet-reopen-per-diagnostic`
- **Finding ID:** CSCAN-elixir_errors-read-snippet-reopen-per-diagnostic
- **Severity:** Low
- **Confidence:** medium
- **Location:** `elixir_errors.erl:218-240`, `read_snippet/2` → `get_file_line/2` → `traverse_file_line/2`
- **Class:** Repeated work
- **User-visible outcome:** Each `read_snippet`-enabled diagnostic re-opens the source and re-reads from the top to the target line: `O(#diagnostics × line-number)` I/O. A large file with many late-line warnings prints them noticeably slower; correctness unaffected.
- **Reproduction:** Compile a file producing N warnings near line L; each triggers a fresh `file:open` + L `read_line` calls.
- **Suggested fix:** Optional — cache the file's lines/offset index per compile job. Only if profiling justifies it.
- **Risk of fix:** memory for the cache; behaviour unchanged.
- **Evidence:** read `elixir_errors.erl:214-240`.

#### `CSCAN-elixir_errors-cut-snippet-nthtail-overrun` — Upgraded from hedged assumption in first pass
- **Finding ID:** CSCAN-elixir_errors-cut-snippet-nthtail-overrun
- **Severity:** Low
- **Confidence:** low
- **Location:** `elixir_errors.erl:455`, `cut_snippet/3`
- **Class:** Crash on stray/unexpected input (off-by-one / index overrun)
- **User-visible outcome:** If the reported error `Line` (from `Location`) is greater than `StartLine + number-of-lines-in-InputString`, then `lists:nthtail(Line - StartLine, Lines)` is asked to drop more elements than the list has and raises `function_clause`. That converts an intended `SyntaxError`/`TokenMissingError` (with a nice snippet) into an opaque internal crash of the parse call.
- **Executed evidence (mechanism):** `lists:nthtail(5, [a,b,c])` → **`error:function_clause`** (ran in Erlang/OTP 29). So an over-count is a hard crash, not a clamp. The guard math is `Line - StartLine`; for `end_line` snippets `cut_snippet(Input, Line, EndLine - Line + 1)` (line 435) also feeds `Line` straight through.
- **Reachability — NOT executed:** whether the parser can emit a `Location` whose `line` exceeds the line count of the `Input` string (e.g. a `TokenMissingError` reported "past" the last content line at EOF) requires driving the real `elixir_tokenizer`→`elixir_parser` (yecc-generated, not in this folder) with a built Elixir. This checkout has no built beam/ebin and no `.ex` sources, so I could not construct the triggering parse. The first pass assumed `elixir.erl` guarantees the line is in-range; I did **not** confirm that, hence a low-confidence finding rather than a dismissal.
- **Suggested fix:** clamp the drop count, e.g. `lists:nthtail(min(Line - StartLine, length(Lines) - 1), Lines)` with an empty-list guard, or fall back to `nil` snippet when the index is out of range.
- **Risk of fix:** low; only affects the degenerate out-of-range case.
- **Evidence:** read `elixir_errors.erl:428-457`; executed `lists:nthtail/2` overrun.

#### `CSCAN-elixir_errors-format-snippet-exotic-position-badarith` — Upgraded from hedged assumption in first pass
- **Finding ID:** CSCAN-elixir_errors-format-snippet-exotic-position-badarith
- **Severity:** Low
- **Confidence:** low
- **Location:** `elixir_errors.erl:122-125` (`format_snippet/6` clause 3 → `get_line_number_digits/2` at 208-210)
- **Class:** Crash on stray/unexpected input (un-spec'd export)
- **User-visible outcome:** A direct caller of the exported `format_snippet/6` passing `Position = none` or `{nil, Col}` together with a non-`nil` `Snippet` reaches clause 3, where `extract_line/1` yields `none`/`nil`, and `get_line_number_digits(none, 1)` / `(nil, 1)` raises `badarith` instead of formatting a diagnostic.
- **Executed evidence (mechanism):** replicating `extract_line` + `get_line_number_digits`: `el(none)=none`, `el({nil,3})=nil`; `get_line_number_digits(none,1)` → **`error:badarith`**, `(nil,1)` → **`error:badarith`**, `(0,1)` → `1` (fine). Confirmed in Erlang/OTP 29.
- **In-module safety (traced):** every in-module path to a non-`nil` Snippet goes through `read_snippet/2`→`get_file_line/2`, which returns a non-`nil` line only when `is_integer(LineNumber), LineNumber > 0` (222); a `none`/`{nil,_}` Position yields a `nil` Snippet and lands on clause 1/2. So the invariant "non-nil Snippet ⟹ integer positive line" holds for `print_diagnostic`/`print_diagnostics`/`print_warning`.
- **Reachability — NOT executed:** whether any *external* caller invokes the exported `format_snippet/6` with such a Position. Grep over the available `src/` finds no caller outside this file, and the callgraph in-folder callers do not call it — but the `.ex` stdlib callers (e.g. `String.Tokenizer.Security`, `Kernel.ParallelCompiler`) are absent from this checkout, so I could not enumerate all callers. Hence low confidence.
- **Suggested fix:** add a `format_snippet` guard/clause treating a non-integer line as the `nil`-Snippet case, or `-spec` the export and normalize Position at the boundary.
- **Risk of fix:** low.
- **Evidence:** read `elixir_errors.erl:107-157,208-231`; executed `get_line_number_digits` with `none`/`nil`/`0`.

#### `CSCAN-elixir_errors-format-error-malformed-utf8-badarg` — Upgraded from hedged assumption in first pass
- **Finding ID:** CSCAN-elixir_errors-format-error-malformed-utf8-badarg
- **Severity:** Low
- **Confidence:** low
- **Location:** `elixir_errors.erl:77` (`emit_diagnostic`) and `elixir_errors.erl:109,120,157,182` (`format_snippet`/`format_message`)
- **Class:** Crash on stray/unexpected input (malformed UTF-8), **not** silent corruption
- **User-visible outcome:** If a `Module:format_error/1` implementation returns malformed UTF-8 (invalid byte sequence), `emit_diagnostic` line 77 `unicode:characters_to_binary(Message)` returns an `{error, Binary, Rest}` tuple rather than a binary; that tuple is stored as the diagnostic's `message` and then flows into `format_snippet`, where `unicode:characters_to_binary([prefix, " ", Message])` (or `io_lib:format("~ts", ...)`) raises `badarg`, crashing diagnostic emission.
- **Executed evidence (mechanism + reclassification):** `unicode:characters_to_binary(<<192,193,255>>)` → `{error,<<>>,<<...>>}`. Feeding that tuple into `unicode:characters_to_binary([<<"warning:">>, " ", Tuple])` → **`error:badarg`**; `io_lib:format("~ts", [Tuple])` → **`error:badarg`**. So the outcome is a hard crash — this refutes the first pass's "silent corruption / fake message" framing: no corrupted binary reaches output. (This is why the *silent-corruption* and *encoding-mishandling* ruled-out entries below are now discharged **by execution**.)
- **Reachability — NOT executed:** whether any real `Module:format_error/1` (in `elixir_expand`, `elixir_module`, `elixir_def`, etc. — several not present in this partial checkout) actually returns non-UTF-8 bytes. In-tree messages are literals/valid chardata, so this is presumed unreachable; low confidence because I could not enumerate all `format_error/1` implementations.
- **Suggested fix:** validate/replace on the `emit_diagnostic` boundary (`unicode:characters_to_binary(Message, utf8, utf8)` with error handling, or `characters_to_binary` result-match), so a bad message degrades to a placeholder instead of crashing the compile job.
- **Risk of fix:** low.
- **Evidence:** read `elixir_errors.erl:61-96,107-183`; executed malformed-UTF-8 through `characters_to_binary` and the two downstream sinks.

## Risks / caveats
- The three crash findings above have **executed-confirmed mechanisms** but **un-executed trigger reachability** — driving the real parser (`nthtail`), enumerating external `format_snippet/6` callers, and enumerating all `format_error/1` implementations all require a built Elixir and the `.ex`/sibling sources absent from this checkout. They are correctly low-confidence, not dismissals: a later pass with a full build should try the reachability directly.
- `format_snippet/6` remains exported without a `-spec`.

### Ruled out

**Operator / precedence typo**
- Reason: Guards (`Module /= nil, LT /= nil` 302; `EndLine > StartLine` 163; `is_binary(ErrorPrefix), is_binary(ErrorSuffix), is_binary(Token)` 385) use `,` (conjunction) intentionally; no `;`/`,` or `andalso`/`orelse` confusion; `Keyword == <<...>>` compares binaries correctly with `==`.
- Axes examined: same-line span, cross-line span, nil span, tuple-vs-atom Error | Not examined: none.
- Basis: verified

**Missing clause / catch-all**
- Reason: `parse_error/5` catch-all (399) covers all binary Error/Token; `format_snippet/6` covers nil/non-nil File and Snippet; `calculate_span_length/2` third clause `#{}` catches remaining maps incl. `span => nil`. Remaining gaps (chardata File, tuple File) are the reported spec findings.
- Axes examined: empty binary Token, `[`/`$`/`{sigil` prefixes, tuple Error, nil span, non-2-tuple span, empty diagnostic list | Not examined: none.
- Basis: verified

**Wrong atom / wrong symbol**
- Reason: Map keys (`span`, `stacktrace`, `details`, `typing_traces`, `severity`, `position`, `file`, `message`) and option keys (`read_snippet`, `span`, `column`, `line`, `end_line`, `error_type`) are used consistently between writer (`emit_diagnostic`) and readers.
- Axes examined: keyfind keys, map-pattern keys, prefix atom names | Not examined: none.
- Basis: verified

**State / accumulator threading error**
- Reason: The `elixir_code_diagnostics` process-dict accumulator (82-94) threads `{[New | Tail], Flag}` correctly; `{Tail, true}` prepends the *printed* diagnostic, `{Tail, false}` the raw one — matching the boolean's meaning.
- Axes examined: undefined dict, `{_,true}`, `{_,false}` branches | Not examined: concurrent mutation (single-process dict, N/A).
- Basis: verified

**Off-by-one / wrong arithmetic (arithmetic axes)**
- Reason: `get_line_number_digits` (1 for <10, +1 per decade), `format_line` column remap (`Column - (SpacesMatched-22)` matches the 3+19 prefix), `cut_snippet` `Line - StartLine` / `Span - 1` framing, and `max(...,0)` guards all check out on the arithmetic axes below. The **`nthtail` overrun** axis is no longer here — it is promoted to a low-confidence finding (`CSCAN-…-nthtail-overrun`) since its reachability could not be executed.
- Axes examined: line 1/9/10/100, deep leading-whitespace trim, column inside trimmed region, empty span-name (Length 0) | Not examined: none (the previously-listed nthtail overrun is now a finding, not a deferred axis).
- Basis: verified

- **Contract violation** — covered as findings above (erl_warn, parse_error spec gaps).

**Crash on stray / unexpected input**
- Reason: The three residual crash mechanisms are now promoted to findings (`nthtail` overrun, `format_snippet/6` exotic Position, malformed-UTF-8 `badarg`), each with executed mechanism evidence. No further un-promoted crash path remains: reachable in-module paths normalize Position/File before formatting, and the non-nil-Snippet⟹integer-line invariant was traced.
- Axes examined: Position `{Line,Col}`/`{0,_}`/`{Line,nil}`/integer/`none`/`{nil,_}` through `extract_line`+`get_line_number_digits` (executed); nil File; nil Snippet; nthtail over-count (executed) | Not examined: none as a dismissal — the residual *trigger reachability* lives in the three findings, not here.
- Basis: verified (as a dismissal of any *additional* crash; the identified crashes are findings)

**Resource leak on error path**
- Reason: `get_file_line/2` opens the IoDevice and closes it via `ok = file:close(IoDevice)` on success; the `{error,_}` open branch acquires nothing. `traverse_file_line` only calls `file:read_line` (returns tuples, does not raise).
- Axes examined: open failure, eof before target line, target line found | Not examined: none.
- Basis: verified

**Silent corruption**
- Reason: The only path where a non-binary could masquerade as a "message" is malformed input to `unicode:characters_to_binary` (77, 109, 120, 157, 182). Executed: a malformed message produces an `{error,_,_}` tuple that is **not** silently emitted — the downstream `characters_to_binary`/`io_lib:format` raises `badarg` first (see finding `…-malformed-utf8-badarg`). So there is no silent-corruption outcome; the failure mode is a crash, captured as a finding.
- Axes examined: valid ASCII/UTF-8; improper-list chardata; **deliberately malformed UTF-8 `<<192,193,255>>` through `emit_diagnostic` line 77 and both downstream sinks (executed, Erlang/OTP 29)** | Not examined: none.
- Basis: verified

**Infinite loop / hang**
- Reason: `traverse_file_line` decrements N to base case 1 (call site guards `LineNumber > 0` integer); `pad_line`, `indent_n`, `get_line_number_digits`, `trim_line` recurse on strictly shrinking structure. No `receive`/loop.
- Axes examined: N=1, N large, eof mid-traverse, empty/blank lines in pad_line | Not examined: none.
- Basis: verified

**Quadratic-where-linear**
- Reason: See the `read_snippet` repeated-work finding; no `Acc ++ X` or per-iteration `length/1` inside loops (accumulators use `[H | recurse]`).
- Axes examined: pad_line/indent_n cons pattern, per-diagnostic file scan | Not examined: none.
- Basis: verified

- **Repeated work** — reported (`read_snippet` reopen).

**Dead / unreachable code**
- Reason: The `{binary(),binary()}` File spec branch and `print_diagnostics([])` are effectively unreachable given current callers, captured under findings/caveats.
- Axes examined: tuple File, empty diagnostics list | Not examined: none.
- Basis: verified

**Documented behaviour mismatch**
- Reason: Module doc says "not exposed in the Elixir language"; functions behave per `-spec` except the two narrowing gaps reported; `prefix/1` returns documented prefixes; `raise/3` builds the exception with `tl/1` dropping the synthetic `throw` frame.
- Axes examined: warning/error/hint prefix, raise stacktrace shape | Not examined: none.
- Basis: verified

**Race condition / shared mutable state**
- Reason: All mutable state is the caller's own process dictionary (`elixir_code_diagnostics`, `elixir_compiler_*`), single-process; no ETS/`system_flag`/shared-table writes here.
- Axes examined: process-dict read/modify/write | Not examined: none.
- Basis: verified

**Wrong scope / order of operations**
- Reason: `meta_location`/`env_format` compute File/Position before use; `emit_diagnostic` reads compiler-file dict before building the map; no side-effect precedes its input validation.
- Axes examined: meta_keep hit/miss, column present/absent | Not examined: none.
- Basis: verified

**Encoding / unicode mishandling**
- Reason: Text is handled as chardata via `unicode:characters_to_binary`/`elixir_utils:characters_to_binary`; `string:length` (not `byte_size`) for display widths (200); tabs counted as width 8 in `trim_line`. The one residual concern — malformed UTF-8 — was executed: it does **not** silently mis-encode; it raises `badarg` (captured as the malformed-UTF-8 crash finding). So there is no silent encoding-corruption outcome.
- Axes examined: multibyte binary line, tab-indented line, improper-list chardata message, **malformed UTF-8 into `characters_to_binary` and downstream `~ts` (executed)** | Not examined: none.
- Basis: verified

### Did I verify this?

- **Operator/precedence, Missing clause, Wrong atom, State threading, Resource leak, Infinite loop, Quadratic, Repeated work, Dead code, Documented behaviour, Race, Wrong scope** — `verified` (unchanged): each cited path read; `Not examined` lists are `none` (state confined to this file; enumerable constructor/guard axes traced).
- **Off-by-one** — now split. The arithmetic axes stay `verified` with `Not examined: none`. The `nthtail` overrun axis, previously `asserted` with `Depends on: elixir.erl`, is **promoted to a low-confidence finding**: I executed the mechanism (`lists:nthtail(5,[a,b,c])` → `function_clause`) but could not execute the reachability (no built parser in this checkout). Per the rule that reading a neighbour cannot clear an axis, and that the axis could not be *tried*, it does not stay a dismissal.
- **Crash on stray input** — previously `asserted`. Executed the exact untried axis (`Position = none`/`{nil,_}` → `get_line_number_digits` `badarith`) and traced the in-module non-nil-Snippet⟹integer-line invariant. The crash mechanism is real, so it is **promoted to a low-confidence finding** (un-spec'd export); no additional un-promoted crash path remains, so the dismissal of *further* crashes is `verified`.
- **Silent corruption / Encoding** — previously `asserted` with `Depends on: Module:format_error/1`. Executed the malformed-UTF-8 axis end to end: the outcome is `badarg` (a crash), **not** silent corruption or mis-encoding. Both class dismissals are therefore now `verified` with `Not examined: none`, and the crash they expose is captured as the malformed-UTF-8 finding.

### Dropped in re-review

- None. All three first-pass findings are preserved. The two first-pass *dismissals* that carried `Depends on:` hedges (silent corruption, encoding) were not dropped but strengthened to `verified` via execution; the three first-pass hedged crash assumptions (nthtail, exotic Position, malformed UTF-8) were promoted to low-confidence findings rather than left asserted.
