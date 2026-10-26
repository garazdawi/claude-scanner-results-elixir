# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_errors.erl
<!-- sha256: e77a5ff61923393aed228beae7e5fe928b188b2962692c62fd5d115b67da3de7 -->

_Analysed on 2026-08-27T17:09:58Z_

All edge inputs produce clean, documented exceptions (no BEAM crash): invalid UTF-8 raises `UnicodeConversionError`, control bytes raise `SyntaxError`, and `cut_snippet`/the sigil path never badmatch. My two crash candidates are robust. The one solid finding is the unsanitized control-character (ANSI/ESC) passthrough into terminal diagnostics, reproduced on both the compile and the parse-only path.

Let me write the report.

---

## Summary

`elixir_errors.erl` renders compiler diagnostics; its one real defect is that source-derived snippet bytes (including raw ESC/ANSI and other C0 control characters) are written to the terminal via `io:put_chars` with no sanitization, reachable from the documented `Code.string_to_quoted/2` API on a parse-only path — a diagnostic-output/terminal-injection issue in the same threat class as the bidi guard the tokenizer already enforces.

## Security

#### `CSCAN-elixir_errors-diagnostic-terminal-escape-injection`
- **Finding ID** — `CSCAN-elixir_errors-diagnostic-terminal-escape-injection`
- **Severity** — Low
- **Confidence** — high
- **Reachability** — reachable
- **CVSS** — `CVSS:3.1/AV:L/AC:L/PR:N/UI:R/S:U/C:N/I:L/A:N` → 3.3 (Low)
- **Location** — `elixir_errors.erl:58` (`print_diagnostic/2`, the `io:put_chars(standard_error, ...)` sink), fed by `read_snippet/2` (line 218/235), `format_line/1` (line 166) and `format_snippet/6` (line 122-157); also reachable via the `io:put_chars` at line 41 (`print_diagnostics/1`) and line 27 (`print_warning/3`).
- **Class** — Template / interpolation (terminal/log-injection of un-escaped control bytes; CWE-150 / CWE-116).
- **Attack scenario** — The attacker is a supplier of an untrusted `.ex` file (a PR author, a published-package author, or a cloned-repo author). The victim is a developer or CI job that runs a *parse-only* tool — a language server, linter, `mix format --check-formatted`, a doc/AST tool, or any code calling `Code.string_to_quoted/2` — over the file and views the tool's diagnostics in a terminal or CI log. The attacker puts a parse-time-warning trigger on a line (e.g. a single-quoted charlist `'x'`, a deprecated `?\`, or an ambiguous operator) and places raw ANSI/cursor-control escape bytes (e.g. `ESC[2K`, `ESC[A`, color codes) elsewhere on that same line, typically in a trailing comment. When the tool parses the file, `elixir_errors` reads that raw source line back from disk and emits it verbatim to stderr. The attacker's escapes then execute in the victim's terminal: they can erase or overwrite preceding output (hiding a real warning about their own malicious code), forge a fake "0 issues / passed" line, or otherwise manipulate what a human reviewer sees. **New capability:** an attacker who can only submit a file to a parse-only pipeline — which otherwise never runs their code and never writes to the victim's terminal — gains control over the victim's terminal/log display and can spoof diagnostic output.
- **Trace** — `Code.string_to_quoted/2` → `elixir:string_to_quoted/5` → on a successful tokenize it calls `emit_warnings/3`, which for each tokenizer/parser warning calls `elixir_errors:erl_warn/3` (line 246-249) → `emit_diagnostic(warning, Location, File, Warning, [], [{read_snippet, true}])` (line 61). With no `elixir_code_diagnostics`/`elixir_compiler_info` in the process dictionary (the plain `Code.string_to_quoted` case), `emit_diagnostic` calls `print_diagnostic(Diagnostic, true)` (line 85). `print_diagnostic` calls `read_snippet(F, P)` (line 46), which opens the caller-supplied `File` and returns the raw line via `traverse_file_line/2` — line 235 only strips the trailing `\n` (`binary:replace(Line, <<"\n">>, <<>>)`) and performs no control-character sanitization. `format_snippet/6` embeds that raw line as `FormattedLine` (via `format_line/1`, which only trims leading spaces/tabs) into the `io_lib:format` output (line 141-155, `~ts`), and `print_diagnostic` writes it with `io:put_chars(standard_error, [Output, MaybeStack, $\n, $\n])` (line 58). The same raw bytes also flow through the compile-time paths (`file_warn/4`, `print_error/4`).
- **Boundary** — Untrusted source-file bytes → the developer's terminal / CI log renderer. The tokenizer is the documented sole sanitizer of source bytes, but (a) it rejects only `?bidi`/`?break` code points, not ESC/other C0 controls (`elixir_tokenizer.erl:1357-1360`, `tokenize_comment`), and (b) `read_snippet` re-reads the line straight from the file, downstream of and bypassing the tokenizer entirely.
- **Impact** —
  - Confidentiality: none directly (the snippet content is the attacker's own file; note that because `read_snippet` reads whatever path is in the diagnostic's `file`, if a caller tokenizes an in-memory buffer while pointing `file:` at a different on-disk file, arbitrary lines of that file are echoed to the terminal — a minor secondary disclosure).
  - Integrity: attacker-controlled terminal/log display — spoofing, hiding, or forging compiler/tool output shown to a human. No integrity impact on files or program behavior.
  - Availability: none beyond nuisance (escape sequences can garble a terminal session).
- **Validation** —
  - Parse-only path (no code execution): `ran /opt/elixir/bin/elixir parse_inj.exs` where `parse_inj.exs` calls `Code.string_to_quoted(File.read!("/tmp/parse_inj.ex"), file: "/tmp/parse_inj.ex", columns: true)` on a file whose line 1 is `x = 'abc' #<ESC>[31mPARSE-ONLY-INJECTION<ESC>[0m`. Observed stderr (via `cat -v`): `1 │ x = 'abc' #^[[31mPARSE-ONLY-INJECTION^[[0m`, and `od` counted 2 raw `1b` (ESC) bytes in the captured stderr.
  - Compile path: `ran /opt/elixir/bin/elixirc esc_test.ex` with a raw ESC in a comment on an unused-variable line → stderr contained `3 │  unusedvar = 1 #^[[31mINJECTED-RED-TEXT^[[0m` with 2 raw ESC bytes.
  - The installed compiler is byte-identical to the file under review (`ran diff /opt/elixir/.../elixir_errors.erl <source>` → "WHOLE FILE IDENTICAL"), so the reproduction reflects this exact code. No guard sanitizes the snippet: `read_snippet`/`traverse_file_line`, `format_line/1`, and `format_snippet/6` were read and none strip/escape control bytes; `highlight/2` only colorizes the `warning:`/`error:` prefix, not the snippet, and fires regardless.
- **Suggested fix** — Before embedding the snippet, sanitize non-printable control characters for display: in `read_snippet/2` (or `format_line/1`) replace C0 controls and DEL (and ESC/CSI specifically) other than `\t` with a visible/escaped form (e.g. Unicode replacement or `\xHH`), mirroring the rejection the tokenizer already applies to `?bidi`/`?break`. Applying it in `format_line/1`/`format_snippet/6` covers every `io:put_chars` sink (lines 27, 41, 58) and the exception-`snippet` field consumed by the raising path.
- **Evidence** —
  - `read /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_errors.erl:25-58,122-240` — `io:put_chars` sinks and the unsanitized `read_snippet`/`format_line`/`format_snippet` chain.
  - `read /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_tokenizer.erl:1357-1362` — comment tokenizer rejects only `?bidi`/`?break`, letting ESC through.
  - `read /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir.erl:481-516` — `emit_warnings` → `erl_warn` wiring from `string_to_quoted`.
  - `ran /opt/elixir/bin/elixir parse_inj.exs` → `1 │ x = 'abc' #^[[31mPARSE-ONLY-INJECTION^[[0m` (2 raw ESC bytes in stderr).
  - `ran /opt/elixir/bin/elixirc esc_test.ex` → `3 │  unusedvar = 1 #^[[31mINJECTED-RED-TEXT^[[0m` (2 raw ESC bytes in stderr).

## Risks / caveats

- **Severity rationale.** This is a diagnostic-channel/terminal-injection issue, not memory corruption, code execution, or an auth bypass — hence Low. Its uplift is real but bounded (display spoofing to deceive a human/CI viewer), and it requires the victim to run a parsing/compiling tool on attacker source and view the output in an escape-interpreting terminal. It is, however, the same threat class (source-text-driven display deception) that Elixir explicitly defends against for bidirectional controls (Trojan Source); the snippet-rendering path simply does not extend that defense to ESC/other C0 controls and, by re-reading the file, sits downstream of the tokenizer's guard.
- **Minor consistency nit (not filed as a finding).** On the raising path, invalid-UTF-8 source makes `cut_snippet`'s `elixir_utils:characters_to_binary/1` raise `UnicodeConversionError` rather than a `SyntaxError`/`TokenMissingError` (`ran` fuzz: `invalid_utf8_mid => {:rescued, UnicodeConversionError}`). It is a clean, documented, catchable exception on genuinely-invalid input, so at most it surprises a caller that rescues only the syntax-error family; no crash of the BEAM.
- **Cross-file dependence.** The finding's reachability relies on `elixir.erl`'s `emit_warnings/erl_warn` wiring (read) and on `elixir_tokenizer.erl` emitting parse-time warnings while not rejecting ESC (read). The ruled-out items below that remain `asserted` name the specific contract each depends on.

### Ruled out

- **`parse_erl_term/1` — `erl_scan:string`/`erl_parse:parse_term` on the parser's offending-token binary (line 408-416), incl. the sigil hard match `{ok,{sigil,_,_,[Content|_],_,_,_}} = ...` (line 358) and the `[<<...` binary clause (line 378).**
  Reason — the token binary is the `~tp` render of a real Elixir token (via `yecctoken2string1/1`'s catch-all, `elixir_parser.erl:883`), whose only element types are atoms/integers/binaries/lists/tuples; Erlang's `~tp` always emits valid term syntax for those, so `parse_erl_term` round-trips and the 7-tuple/non-empty-`Parts` shape always holds. Empty sigils yield `Parts=[<<>>]` (non-empty), and re-scanning creates no atoms that tokenization didn't already intern.
  Axes examined: empty sigil (`Parts=[<<>>]`), interpolation parts, quotes/apostrophes/backslash, NUL+high bytes, embedded newline/CRLF, unicode (BMP+astral), modifiers, uppercase sigil name, brace/tuple-looking content, 5000-byte content, `static_atoms_encoder` non-atom `Atom` (handled by `is_atom/1`, line 365) | Not examined: none.
  Basis — verified (`ran trace4.erl`: all 13 adversarial cases `parse_erl_term=ok hardmatch=matched`; `ran trace.erl` confirmed empty-sigil `Parts=[<<>>]`).

- **`cut_snippet/2,3` → `lists:nthtail(Line - StartLine, Lines)` and `[Head|Tail] = ...` (line 449-457).**
  Reason — the tokenizer increments its line counter only on `\n`/`\r\n` (`elixir_tokenizer.erl:659-663` → `tokenize_eol` → `Line+1`), and `cut_snippet` splits on `"\n"`, so both key off the same newline; the reported error line is always ≤ `StartLine + (number of \n)` = `StartLine + length(Lines) - 1`, keeping the `nthtail` index in `[0, length(Lines)-1]`. Fuzzing incomplete/mismatched/multi-line/CR/VT/FF/unicode-separator inputs produced only clean `SyntaxError`/`TokenMissingError`/`MismatchedDelimiterError`, never a badmatch/function_clause.
  Axes examined: bare CR, CRLF, trailing newlines, VT/FF, LS/PS, deeply nested/incomplete, mismatched delimiters (with `end_line`), heredocs, `StartColumn`/`Indentation` prelude, negative index (Line<StartLine) | Not examined: none.
  Basis — verified (`ran probe.exs`, `probe2.exs`, `fuzz.exs`; all rescued to the syntax-error family; line-counting parity read at `elixir_tokenizer.erl:659-663,755-759`).

- **`get_file_line/2`/`traverse_file_line/2` — `file:open(File,[read,binary])` + N-line `file:read_line` loop (line 222-240).**
  Reason — `File` is the diagnostic's `file` (compiler-chosen, or the caller's own `:file` option — not an attacker-supplied *path*, and `nil`/`<<"nofile">>` short-circuit to `nil`); no external path components are joined, so no traversal. The O(N) read loop's N is the diagnostic line number, which for tokenizer-derived positions is bounded by the file's own newline count (content alone cannot make line ≫ file length); a huge N requires the *caller* to pass an out-of-range `line:` start, which is self-inflicted.
  Axes examined: nil/"nofile" sentinels, path origin (caller/compiler vs attacker), huge line number vs file length, single-line read+close (no persistent handle) | Not examined: symlink/FIFO target of the caller's `:file` path (caller-chosen, out of scope).
  Basis — asserted.
  Depends on: `elixir.erl` — the `:file`/position in a diagnostic originates from the compiler or the API caller's own options, and tokenizer-assigned line numbers do not exceed the source's newline count.

- **`Module:format_error(Desc)` dynamic dispatch (line 260, 293).**
  Reason — `Module` is an error-formatting module reference passed by internal compiler callers (`elixir_expand`, `elixir_module`, `elixir_dispatch`, …), never derived from external bytes; `Desc`→string is a formatting operation, not a sink for arbitrary terms.
  Axes examined: origin of `Module` (internal callers per callgraph), origin of `Desc` (compiler passes) | Not examined: none.
  Basis — verified (module argument is supplied literally by in-folder callers listed in the callgraph slice; no external-data path reaches the `Module` position).

- **`raise/3` → `Kind:exception([...])` dynamic dispatch (line 529).**
  Reason — every caller passes a literal atom `Kind` (`'Elixir.CompileError'`, `'Elixir.SyntaxError'`, `'Elixir.TokenMissingError'`, `'Elixir.MismatchedDelimiterError'`); `Kind` is never computed from input.
  Axes examined: all `raise`/`raise_snippet`/`raise_reserved` call sites (lines 312, 315, 327, 331, 343, 354, 374, 382, 387, 396, 403, 405, 419, 426) | Not examined: none.
  Basis — verified (`read` of every call site; all `Kind` values are literals).

- **`unicode:characters_to_binary(Message)` (line 77) and downstream `~ts` on `Message`.**
  Reason — `Message` is `Module:format_error(Desc)` output or a tokenizer warning string; tokens are validated UTF-8 by the tokenizer (invalid UTF-8 is rejected before a message is built), so `Message` is valid chardata and the call returns a binary rather than an `{error,_}`/`{incomplete,_}` tuple.
  Axes examined: invalid UTF-8 in source (tokenizer-rejected), developer format strings | Not examined: a caller/pass deliberately passing invalid chardata as `Desc` that survives `format_error`.
  Basis — asserted.
  Depends on: `elixir_tokenizer.erl` — surfaces invalid UTF-8 as an error (UTF8_DECODE_ERRORS_SURFACED), so message text derived from tokens is valid chardata.

- **Process-dictionary state in `emit_diagnostic` (`elixir_code_diagnostics`, `elixir_compiler_info`, `elixir_compiler_file`, line 71-94) and diagnostic-list accumulation.**
  Reason — these keys are compiler-internal, per-process routing state set by the compilation driver; they are not attacker-controlled across a trust boundary, and the accumulated list length is bounded by the finite number of diagnostics a given source produces.
  Axes examined: key origin (compiler driver, not input), list growth bound (per-source diagnostic count) | Not examined: none.
  Basis — verified (`read` of `emit_diagnostic`; all three keys are compiler-owned and none is populated from the reviewed API's untrusted arguments).

### Did I verify this?

- `parse_erl_term` round-trip / sigil hard match — **verified**; `Not examined: none`. Directly executed the exact `~tp`-then-`parse_erl_term`-then-hard-match sequence on 13 adversarial sigil tokens plus the empty-sigil `Parts` shape; no `Depends on:` needed.
- `cut_snippet` nthtail/badmatch — **verified**; `Not examined: none`. Line-count parity read in the tokenizer and exercised across CR/VT/FF/LS/PS/CRLF/nested/mismatched/heredoc/prelude inputs; no crash observed.
- `Module:format_error` dispatch — **verified**; `Not examined: none`. The `Module` argument is supplied literally by the in-folder callers named in the callgraph slice, with no external-data path to that position.
- `raise`→`Kind:exception` — **verified**; `Not examined: none`. Every call site read; all `Kind`s literal.
- Process-dict state — **verified**; `Not examined: none`. Keys are compiler-owned; not reachable from untrusted API arguments.
- `get_file_line`/`traverse_file_line` file read + O(N) loop — **asserted**, retained `Depends on: elixir.erl` (diagnostic `file`/`line` originate from the compiler or caller options, and tokenizer line numbers are bounded by source newlines). The residual untried axis is the symlink/FIFO nature of a *caller-chosen* `:file` path, which is out of scope (self-inflicted), so I leave this asserted rather than manufacture an out-of-scope repro.
- `unicode:characters_to_binary(Message)` — **asserted**, retained `Depends on: elixir_tokenizer.erl` (invalid UTF-8 surfaced before message construction). I did not construct a public-API input that forces invalid chardata into `Message` without first being rejected by the tokenizer; the dependency is the honest boundary and is flagged for the cross-file pass.

