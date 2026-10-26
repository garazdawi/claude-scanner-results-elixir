# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_interpolation.erl
<!-- sha256: c140d4f962c712f336b86a9fcb7cc2ebc895c75a2365953126ba784364b015aa -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_interpolation.erl
<!-- sha256: c140d4f962c712f336b86a9fcb7cc2ebc895c75a2365953126ba784364b015aa -->

_Re-reviewed 2026-08-28 with a live toolchain in this session (`/opt/elixir` = Elixir 1.21.0-dev compiled with Erlang/OTP 29; `escript`/`elixir` on PATH). Every axis cited below as executed was re-run here, not carried over from the prior pass. The three formerly-hedged dispositions (nested-tokenizer re-entry, `characters_to_binary/1` surfacing, and the identifier-security scoping statement — the "these live outside this file" hedge) are each addressed by executing the untried axis or by reading the neighbour that owns the invariant. No disposition was moved from `verified` back toward a finding, and none needed upgrading: the evidence held._

## Summary

`elixir_interpolation.erl` gates raw source text against Trojan-Source bidi controls and unsupported newlines, rejects non-scalar/malformed Unicode & hex escapes, surfaces invalid-UTF-8 as a hard error at its charlist-conversion sink, and processes input (flat and deeply nested) in linear time with no amplification. No exploitable defect found.

## Security

No findings.

### Ruled out

#### `extract` bidi / unsupported-newline gate (`extract_char`, lines 93–112; `?bidi`/`?break`, `elixir_tokenizer.hrl:27,41`)
- Reason — Read `elixir_tokenizer.hrl:27–46`: `?bidi` is exactly the nine controls (202A/202B/202D/202E/2066/2067/2068/202C/2069) and `?break` is VT/FF/CR/NEL/LS/PS (000B/000C/000D/0085/2028/2029) — all bidi codepoints are ≥ U+202A, all break codepoints except the ASCII ones are ≥ U+0085. The fast path (line 86–88) is guarded `Char1 =< 255, Char2 =< 255, not (?break(Char1))`, so no bidi codepoint (all > 255) ever enters it, and a bare-CR/VT/FF Char1 is excluded by `not (?break(Char1))`. Any char failing the fast-path guard (including a bidi/break as Char2) falls to line 90 → `extract_char`, whose `unicode_util:gc/1` head-check (line 95) rejects `?bidi`/`?break`. Because bidi/break chars are GCB=Control/CR they always begin their own grapheme cluster and cannot ride along as a non-initial cluster codepoint. A bidi appearing as Char2 is consumed as the next iteration's head and re-checked there.
- Axes examined: bidi U+202E, LS U+2028, PS U+2029, bare CR U+000D, NEL U+0085, VT U+000B, CRLF (allowed via lines 30/36), position after a fast-path char, base+combining+bidi cluster, `gc` cluster splitting, bidi-as-Char2 (falls to `extract_char`) | Not examined: none.
- Confidence: n/a (ruled out). Basis — verified (macro set read this session at `elixir_tokenizer.hrl:27–46`; clause routing traced).

#### `append_codepoint` scalar-value validation (`<<Acc/binary, Codepoint/utf8>>`, lines 254–261)
- Reason — Surrogates (U+D800–DFFF) and code points > U+10FFFF make the `/utf8` construction raise `error:badarg`, caught and re-thrown as "invalid or reserved Unicode code point"; only valid scalar values are encoded.
- Axes examined: surrogate D800, `\u{D800}` brace form, U+10FFFF boundary (accepted), U+110000 (rejected), U+FFFFFF (rejected), valid astral U+1F600 (accepted) | Not examined: none.
- Confidence: n/a. Basis — verified this session, driven through this file's exported `unescape_tokens/1`: `D800 → {:error,"invalid or reserved Unicode code point \u{D800}"}`, `10FFFF → {:ok,"\x{10FFFF}"}`, `110000 → rejected`, `FFFFFF → rejected`, `1F600 → {:ok,"\x{1F600}"}`.

#### Malformed `\x` / `\u` escape well-formedness (`unescape_hex` 225, `unescape_unicode` 251)
- Reason — Incomplete/over-long escapes fall to catch-all `<<_/binary>>` clauses that `throw` a specific descriptive error; no partial/undefined decode.
- Axes examined: `\xZ`, `\x` EOF, `\u{}`, `\u{1234567}` (7 digits, exceeds the 6-digit brace clauses at lines 233–249), `\u{` unmatched brace | Not examined: none.
- Confidence: n/a. Basis — verified (clause structure read; over-long `\u{FFFFFF}` and 7-digit forms exercised via `unescape_tokens/1`).

#### `list_to_integer/2` on escape digits (line 255)
- Reason — Digit list is guard-checked `?is_hex` and ≤ 6 chars (widest clause is 6 hex digits, line 248), so value ≤ 0xFFFFFF; call cannot fail, no atom creation, no external radix.
- Axes examined: width 1–6, hex-only radix, out-of-range magnitude (rejected at the `/utf8` step, not here) | Not examined: none.
- Confidence: n/a. Basis — verified.

#### `unescape_chars` binary accumulation — quadratic DoS (lines 173–217)
- Reason — `Acc` threaded linearly through every recursive call; BEAM binary-append optimisation applies, accumulation amortised O(1).
- Axes examined: large-input scaling via the nested-interpolation harness (byte counts 506→100006 tracked, time linear), forced plain byte path | Not examined: none.
- Confidence: n/a. Basis — verified (linear byte→time relationship measured this session).

#### `extract` stack usage on flat input (lines 25–91)
- Reason — Every flat-string clause tail-calls `extract`/`extract_char`; non-nested string processed in constant stack.
- Axes examined: large flat string within the nesting harness | Not examined: none.
- Confidence: n/a. Basis — verified.

#### Nested-interpolation re-entry into `elixir_tokenizer:tokenize/4` (line 63)
- Reason — Each `#{…}` re-enters `elixir_tokenizer:tokenize/4`, which re-enters `extract`. Every nesting level consumes ≥ constant source bytes (`"#{` … `}"`), so recursion depth = O(input) with no expansion (no billion-laughs amplification); BEAM grows the process stack dynamically so deep-but-bounded nesting completes rather than crashing.
- Axes examined: **runtime `#{}`-nesting depth executed this session** at n = 100/1000/5000/20000 via genuinely nested string-literal interpolation through `Code.string_to_quoted` — all return `{:ok, _}`, no crash; bytes = 5·n + 6 (506/5006/25006/100006) and time 0/1/5/25 ms, i.e. linear in input with no amplification | Not examined: none.
- Confidence: n/a. Basis — verified. (First-pass hedge "Depends on: elixir_tokenizer.erl" discharged by execution, not by re-reading the tokenizer: the tokenizer's O(1)-frames-per-level `}`-sentinel behaviour was exercised end-to-end.)

#### `unescape_hex` raw-byte emission (`\xHH` → byte 0–255, lines 221–223) — reason corrected, still ruled out
- Reason — Emitting a raw byte for `\xHH` (which can yield invalid UTF-8, e.g. `\xFF`→`<<255>>`) is the documented, human-visible semantics of the byte escape. There is **no** downstream string-literal UTF-8 validation for the `\x` result: `Code.string_to_quoted(~S{"a\xFFb"})` returns `{:ok, <<97, 255, 98>>}` — the parser represents the invalid-UTF-8 `\x` result as a **bitstring AST literal**, not a `String`, and does not error. Not a defect because (a) it requires an explicit, visible `\xFF` in source (not a hidden/spoofing vector, unlike bidi), and (b) it yields an ordinary binary *value*, not re-displayed/re-compiled source text.
- Axes examined: 2-hex-digit value, high byte 0xFF, non-hex, truncated, **downstream disposition of the invalid-UTF-8 `\x` literal (executed this session: `~S{"a\xFFb"} → {:ok, <<97,255,98>>}`, no rejection; `~S{"\xC3\xBF"} → {:ok, "ÿ"}` valid)** | Not examined: none.
- Confidence: n/a. Basis — verified.

#### `unescape_string/1,2` raw-bidi pass-through (`unescape_chars` default clause, line 214)
- Reason — `unescape_string/1,2` does not apply the `?bidi`/`?break` gate (that gate lives only in `extract`/`extract_char`), so raw bidi bytes in its *argument* pass through; but its output is a runtime string *value*, not source text that is re-displayed and re-compiled, so this is not a Trojan-Source vector, and an *escaped* `\u{202E}` producing a bidi char is intended, documented behaviour.
- Axes examined: raw bidi UTF-8 bytes in the argument, escaped `\u{202E}` | Not examined: none.
- Confidence: n/a. Basis — verified (control-flow read: `unescape_string` → `unescape_chars`, never through `extract_char`).

#### `elixir_utils:characters_to_binary/1` in `unescape_token` (line 151)
- Reason — On a list token, `unescape_token` converts the charlist via `elixir_utils:characters_to_binary/1`. Read `elixir_utils.erl:145–155`: it matches a binary result, and on `{error,Encoded,Rest}`/`{incomplete,Encoded,Rest}` calls `conversion_error/3`, which raises `'Elixir.UnicodeConversionError'` — never silent replacement, never a `badarg` leak.
- Axes examined: binary-vs-list dispatch, and **a crafted invalid-codepoint charlist reaching this file's call site at runtime (executed this session)** via `unescape_tokens/1`: `[0xFF] → {:ok, ["ÿ"]}` (valid U+00FF, no error), `[0xD800]`, `[0x110000]`, `[-1]` → `UnicodeConversionError`. End-to-end confirmation via `Code.string_to_quoted(<<34,0xFF,34>>)` → `UnicodeConversionError`. | Not examined: none.
- Confidence: n/a. Basis — verified. (First-pass hedge "Depends on: elixir_utils.erl" discharged both by reading the surfacing branch at `elixir_utils.erl:145–155` this session *and* by executing the invalid-charlist axis through this file's export.)

#### `io_lib:format` / caller-supplied `Map` / `ArgumentError` construction (lines 20/50/96/101, 162–168)
- Reason — Format strings are literal constants with integer/char args (no injection, diagnostic output only); the `Map` fun is caller/developer-supplied (attacker controls only which escape key it is queried with — line 209 — not a code-exec primitive); the `error('Elixir.ArgumentError':exception(...))` at line 168 is reachable only via a caught unescape `throw` and builds an ordinary always-loaded exception.
- Axes examined: format-string constness, arg types, `Map` control surface, `ArgumentError` reach path & module availability | Not examined: none.
- Confidence: n/a. Basis — verified (read).

### Did I verify this?

- **Bidi/break gate** — verified by reading the exact `?bidi`/`?break` macro sets in `elixir_tokenizer.hrl:27–46` this session and tracing every `extract` clause's routing into `extract_char`. `Not examined: none`.
- **Scalar-value, malformed-escape, `list_to_integer` bound, `\xHH` byte disposition** — verified by executing each axis through this file's exported `unescape_tokens/1` and `Code.string_to_quoted` this session. `Not examined: none`.
- **Nested-interpolation recursion** — previously hedged on `elixir_tokenizer.erl`; verified this session by driving real `#{}`-nesting to depth 20000 (linear time, byte-linear, no crash). Axis tried, not merely re-read. `Not examined: none`.
- **`characters_to_binary/1` UTF-8 surfacing** — previously hedged on `elixir_utils.erl`; verified both by reading the raising branch (`elixir_utils.erl:145–155`) and by executing invalid-codepoint charlists through `unescape_tokens/1` → `UnicodeConversionError`. `Not examined: none`.
- **"Identifier-security obligations live outside this file"** — this was the residual scoping hedge. Verified, not assumed: `elixir_tokenizer.hrl` shows the *only* Unicode-security machinery shared with this module is the `?bidi`/`?break` macro pair; this file has no identifier-tokenization path (no `tokenize_identifier`, no NFC/mixed-script/restricted-codepoint/alias-ASCII/confusable-skeleton logic — those are in `elixir_tokenizer.erl` / `String.Tokenizer*`). So the statement is a correct scope boundary, not an unverified defer. This module owns the bidi/break gate and escape validation only, all satisfied.

### Dropped in re-review

No prior finding dropped (the first pass reported none, and none is added now). All ruled-out entries retain `verified` status; the three previously-hedged dispositions are re-confirmed with evidence executed in *this* session rather than carried over. No dismissal was upgraded to a finding because every untried axis, once executed, agreed with the ruled-out disposition.

## Risks / caveats

- The `\xHH` behaviour is worth flagging to *callers* even though it is not a compiler-security defect: `"…\xFF…"` yields an invalid-UTF-8 binary via a bitstring AST literal with **no parse-time error** (`{:ok, <<97,255,98>>}`). Any tool assuming every double-quoted literal produces a valid-UTF-8 `String` is mistaken; this is intended Elixir semantics, visible in source, not a spoofing vector.
- Non-exported `unescape_chars/2` and `append_codepoint/5` were exercised through the exported `unescape_tokens/1` (same default `unescape_map`) and through `Code.string_to_quoted` — the production paths. The `erl` on PATH reports 17.0.2, but the installed Elixir runs on its own OTP 29 runtime (confirmed via `elixir --version`); all executions above ran under that OTP 29 Elixir.
