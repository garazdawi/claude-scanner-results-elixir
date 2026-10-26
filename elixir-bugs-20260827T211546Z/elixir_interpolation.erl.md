# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_interpolation.erl
<!-- sha256: c140d4f962c712f336b86a9fcb7cc2ebc895c75a2365953126ba784364b015aa -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T06:21:46Z_

# Bug-hunting review — `elixir_interpolation.erl`

- **Summary** — One reachable crash-on-stray-input defect: `extract_char/8`'s `case` over `unicode_util:gc/1` omits the documented `{error, Rest}` return, so an invalid code point in a charlist reaching interpolation extraction raises `case_clause`.

## Bugfixes

#### `CSCAN-elixir_interpolation-gc-error-case-clause`
- **Finding ID:** `CSCAN-elixir_interpolation-gc-error-case-clause`
- **Severity:** Medium
- **Confidence:** medium
- **Location:** `elixir_interpolation.erl:93-112`, `extract_char/8`
- **Class:** Crash on stray / unexpected input
- **User-visible outcome:** When the character stream handed to `extract_char/8` begins with a code point that `unicode_util:gc/1` considers malformed (a value `unicode_util` rejects, e.g. `16#110000` or a lone surrogate embedded in a charlist), `gc/1` returns the tuple `{error, Rest}` rather than a `[Char|Rest]` list or `[]`. None of the four `case` clauses (`[Char|_]` bidi/break, `[Char|_]` list, `[Char|_]` integer, `[]`) match a tuple, so the tokenizer process dies with `case_clause` instead of producing a clean `{error, Reason}` location tuple. A caller of `elixir_tokenizer:tokenize/*` / `elixir:string_to_tokens/5` that passes a charlist containing such a code point inside a string/sigil observes an uncaught `case_clause` exception (crash) instead of the documented `{error, Reason}` diagnostic it gets for every other bad-input case.
- **Reproduction:** Tokenize a quoted string whose content charlist contains an out-of-range code point, e.g. the interpolation content `[16#110000]` inside `"..."`. Because `16#110000 > 255`, the two-char fast path at line 86 fails its `Char1 =< 255` guard and control reaches line 90 → `extract_char/8` → `unicode_util:gc([16#110000|_])`, which returns `{error, _}` → `case_clause`. (For *binary* source input the charlist conversion rejects bad bytes upstream, so this is reached via the charlist-accepting API surface.)
- **Suggested fix:** Add a clause to the `case` in `extract_char/8` handling `{error, _}` — e.g. treat it like the bidi/break error path and return `{error, {?LOC(Line, Column), "invalid encoding starting at", ...}}` (or reuse the existing invalid-character error shape) so the malformed code point yields a located `{error, Reason}` rather than a crash.
- **Risk of fix:** none for well-formed source (that path is unchanged); only changes an otherwise-crash into a structured error.
- **Evidence:** read `elixir_interpolation.erl:93-112` (the four-clause `case` with no tuple clause); read `elixir_interpolation.erl:86-91` (fast path bypassed for code points `> 255`); `unicode_util:gc/1`'s documented return set includes `{error, Rest}`.

## Risks / caveats
- The one finding's severity hinges on a caller passing a charlist (not a binary) containing an invalid code point through the documented tokenizer entry points; for binary source, upstream UTF-8 validation in `elixir_tokenizer`/`elixir_utils` makes it unreachable. I could not execute `unicode_util:gc/1` here to confirm the exact malformed inputs that yield `{error,_}` versus pass-through — hence medium confidence.
- The `<<Acc/binary, Other>>` append at line 211 would silently truncate to 8 bits if a *custom* `Map` returned a value `> 255`; the built-in `unescape_map/1` never does, and a custom map returning a >255 byte violates the map's implied contract, so this is excluded as caller-contract misuse rather than a bug.

### Ruled out

- **Operator / precedence typo**
  - Reason: The one compound guard (line 87, `Char1 =< 255, Char2 =< 255, not (?break(Char1))`) is all-`,` conjunction with intended semantics; `?is_hex`/`?bidi`/`?break` macros use explicit `orelse`/`andalso` and parenthesize correctly.
  - Axes examined: two-Latin1-char path, break char as Char1, bidi char (>255 impossible here), single-char tail | Not examined: none
  - Basis: verified

- **Missing clause / catch-all**
  - Reason: The `unicode_util:gc/1` `case` in `extract_char/8` is missing the `{error,_}` arm — reported as the finding above; all other `case`/function dispatch (`extract/8`, tokenizer-reentry `case` at 63-79, `unescape_*`) have fall-through clauses.
  - Axes examined: gc `{error,_}` return, tokenizer error tuples, empty vs single vs multi-char, heredoc triple terminator, trailing lone backslash | Not examined: none
  - Basis: verified

- **Wrong atom / wrong symbol**
  - Reason: Record field accesses (`#elixir_tokenizer{...}`, `.column`, `.cursor_completion`, `.warnings`) and `unescape_map` atom keys (`newline`/`unicode`/`hex`) match their use sites and the header.
  - Axes examined: scope record fields, map keys, error atoms | Not examined: none
  - Basis: verified

- **State / accumulator threading error**
  - Reason: `Buffer`/`Output`/`Scope` threading is consistent — buffer reset to `[]` after interpolation flush (line 69), original `Buffer` (not the space-accumulated `NewBuffer`) correctly used on the heredoc-terminator branch (line 119), `NewScope` (with updated warnings) forwarded after reentry.
  - Axes examined: interpolation flush, heredoc terminator vs continuation buffer, warning threading across `#{}` reentry, empty-buffer finish | Not examined: none
  - Basis: verified

- **Off-by-one / wrong arithmetic**
  - Reason: Column increments match consumed-char counts (`+2` for `#{`/`\X`, `+3` for `\#{`/heredoc terminator, `+4` for `\"""`, `+1` per grapheme), and `strip_horizontal_space` seeds Counter at 1 to yield 1-based column.
  - Axes examined: `#{`, escaped terminator, escaped triple, escaped `#{`, heredoc dedent column, grapheme-cluster column | Not examined: none
  - Basis: verified

- **Contract violation**
  - Reason: `extract/6` returns the documented `{Line, Column, Tokens, Remaining, Done, Scope}` on success and `{error, Reason}`/`{error, {string,...}}` on failure; `unescape_tokens/1` returns `{ok, List}` or the caught `{error,_,_}`; `unescape_string/*` returns a binary or raises `ArgumentError`.
  - Axes examined: success shape, missing terminator, interpolation-terminator error, cursor completion | Not examined: none
  - Basis: verified

- **Resource leak on error path**
  - Reason: No files, ETS tables, processes, monitors, or counters are created in this module; it is pure list/binary transformation.
  - Axes examined: whole-module scan for side-effecting BIFs | Not examined: none
  - Basis: verified

- **Silent corruption**
  - Reason: Buffers are reversed exactly once at flush (`build_string`, `finish_extraction`), interpolation tokens reversed once (line 68/75), and the empty-string special-case `[]->[[]]` preserves an empty segment rather than dropping it.
  - Axes examined: empty string, leading interpolation, consecutive interpolations, heredoc line, multibyte UTF-8 pass-through in `unescape_chars` | Not examined: none
  - Basis: verified

- **Infinite loop / hang**
  - Reason: Every recursive `extract`/`extract_char`/`unescape_chars` call consumes at least one input element or terminates on `[]`/`<<>>`; no `receive`, no unbounded retry.
  - Axes examined: empty-input termination, single-char progress, gc `[]` base case, unescape binary base case | Not examined: none
  - Basis: verified

- **Quadratic-where-linear**
  - Reason: Buffers use `[X|Acc]` prepend with a single terminal `lists:reverse`; `unescape_chars` uses the compiler-optimized linear `<<Acc/binary, ...>>` append idiom, not `++`.
  - Axes examined: buffer accumulation, binary append, output list building | Not examined: none
  - Basis: verified

- **Repeated work**
  - Reason: No pattern is recompiled per iteration; `binary:match(String,<<$\\>>)` runs once per `unescape_chars/2` entry to skip the leading literal chunk, which is a strict optimization.
  - Axes examined: per-call pattern compilation, redundant scans | Not examined: none
  - Basis: verified

- **Dead / unreachable code**
  - Reason: All `extract` clauses are reachable given clause ordering (escaped-terminator before escaped-triple before `#{` before generic backslash before the two-char fast path before the gc catch-all); the `{ok,...}` cursor branches gate on `cursor_completion`.
  - Axes examined: clause-order reachability, cursor-completion branches, tokenizer error-shape branches | Not examined: none
  - Basis: verified

- **Documented behaviour mismatch**
  - Reason: Comments ("This is called by Elixir", the `\r\n`/`\n` escape example, `?break` header note that call sites must handle CRLF before the break check) match the code: lines 30/33/36 handle CR/LF/CRLF before any `?break` check reaches a bare CR.
  - Axes examined: CRLF-before-break claim, escaped-newline unescaping, escaped-terminator deprecation warning for uppercase sigils | Not examined: none
  - Basis: verified

- **Race condition / shared mutable state**
  - Reason: Pure functional module; no process state, ETS, or system flags touched.
  - Axes examined: shared-state scan | Not examined: none
  - Basis: verified

- **Wrong scope / order of operations**
  - Reason: Buffer is flushed to `Output` (line 62) before the interpolation token is built and before the tokenizer reentry, and warnings are captured from the sub-tokenize result before continuing.
  - Axes examined: flush-before-reentry, warning capture order | Not examined: none
  - Basis: verified

- **Encoding / unicode mishandling**
  - Reason: `unescape_chars` walks the binary byte-wise but only special-cases ASCII `\`, copying multibyte UTF-8 bytes through unchanged; `extract` routes code points `>255` and graphemes through `unicode_util:gc`; bidi/break code points are rejected. (The single gap is the unhandled `gc` `{error,_}` return, captured as the finding.)
  - Axes examined: multibyte UTF-8 pass-through, grapheme cluster (list) handling, bidi/break rejection, byte-vs-codepoint in unescape | Not examined: none
  - Basis: verified

### Did I verify this?

- All ruled-out entries above are marked `verified`: for each I read the governing clauses in this file and traced the triggering input shapes, and each `Not examined` list reads `none`. No `Depends on:` lines remain because the dismissals rest on code within this file.
- The one cross-file dependence is in the **finding itself** (not a dismissal): the crash's reachability depends on `unicode_util:gc/1` returning `{error, Rest}` for invalid code points and on `elixir_tokenizer` forwarding charlist code points to `extract` without prior validation. I could not execute `gc/1` to enumerate exactly which code points trigger `{error,_}`, so the finding is held at medium confidence rather than upgraded — that is the honest limit of what reading (not running) established.

