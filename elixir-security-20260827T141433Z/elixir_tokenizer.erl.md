# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_tokenizer.erl
<!-- sha256: dabb8e67cec4de43adbf387150e9df1d40000fa1cb19f3e9fdbb15b0b4d73b51 -->

_Analysed on 2026-08-27T18:42:49Z_

Analysis complete. I have read the full target file (2018 lines), its header (`elixir_tokenizer.hrl`), the primary callee `elixir_interpolation.erl`, and the relevant parts of `elixir_utils.erl`, and I ran targeted tests on the security-critical primitives (utf8 scalar validation, atom-length `system_limit` boundary, big-integer cost). Here is the report.

---

# Security review — `elixir_tokenizer.erl`

## Summary
The Elixir tokenizer correctly enforces the Unicode/Trojan-Source obligations it owns (bidi/break rejection in comments in-file; strings/sigils and identifiers delegated to `elixir_interpolation` and `String.Tokenizer`), guards every unbounded atom sink behind a length cap and the `existing_atoms_only`/`static_atoms_encoder` options, and validates Unicode escapes via UTF-8 scalar construction — I found **no security defect**.

## Security
No findings.

All security-relevant sinks in this file are either bounded by a finite operator/delimiter alphabet, correctly guarded, correctly delegated to a sibling that performs the check, or represent documented-and-mitigable by-design behavior (atom creation). Details in **Ruled out** and **Risks / caveats**.

## Ruled out

- **`unsafe_to_atom` — unbounded atom creation from identifiers/atoms/quoted-atoms/multi-letter-sigils (CWE-770 atom exhaustion)** — lines 1077–1167, 553, 850, 998, 1805.
Reason: This is the tokenizer's intended function (source identifiers become atoms); it is fully gated — with `existing_atoms_only=true` it routes to `binary_to_existing_atom`/`list_to_existing_atom` (lines 1110, 1140) and with a `static_atoms_encoder` to the caller's encoder (1087–1107), so a caller parsing untrusted input can prevent all unbounded creation. The default (create atoms) is the documented behavior of `Code.string_to_quoted/2`. No defect in this file. See Risks.
Axes examined: existing_atoms_only on/off, static_atoms_encoder present/absent, binary-vs-list input, multi-letter sigil path, quoted-atom path | Not examined: none.
Basis: verified.

- **`unsafe_to_atom` length guard — `system_limit` crash on >255-char atom** — lines 1077–1086.
Reason: The `byte_size(Part) > 255` / `length(Part) > 255` guard fires before any `*_to_atom` call, returning `{error, …}` instead of crashing; it is conservative (byte-count on binaries over-rejects some valid multibyte atoms but never lets an over-long one through).
Axes examined: 256-char list (→ `system_limit` without guard), 255-byte binary (ok), 400-byte/200-char multibyte binary (guard over-rejects, no crash) | Not examined: none.
Basis: verified — `ran erlc/erl`: `list_to_atom/256chars: {error,system_limit}`, `binary_to_atom/255bytes: 255`, `mb bytes=400 chars=200 atom=200`.

- **Bounded `list_to_atom` on operator/delimiter/single-letter-sigil text** — lines 342, 357, 364, 386–521, 435, 439, 444, 1024, 1046, 1522–1523, 1804.
Reason: Each argument is drawn from a fixed finite alphabet (operator glyphs matched by the `?…_op` macros, container punctuation, string delimiters, single ASCII sigil letters), so the set of producible atoms is finite and cannot exhaust the table.
Axes examined: operator length (1–3), delimiter set (`" ' / < > [ ] ( ) { } |`), single-letter sigil (52), interpolation_format opening/closing call sites | Not examined: none.
Basis: verified.

- **Bidi/break rejection in comments** — `tokenize_comment/2`, lines 1353–1364.
Reason: Every comment codepoint is tested with `?bidi`/`?break` before accumulation; CRLF/LF end the comment first (matching the hrl contract), and a bare CR (`?break(16#000D)`) errors. No context bypass (claim BIDI_COMMENT_NO_BYPASS).
Axes examined: all 9 bidi controls, VT/FF/bare-CR/NEL/LS/PS, CRLF-before-break ordering | Not examined: none.
Basis: verified (read 1353–1364 + hrl 27–46; clause order makes every char pass the guard).

- **Bidi/break rejection in strings/sigils/heredocs** — delegated to `elixir_interpolation:extract` → `extract_char`, lines 93–112.
Reason: `extract_char` rejects `?bidi`/`?break` for every codepoint; the ≤255 fast-path (interpolation 86–88) excludes break chars (`not ?break(Char1)`) and all bidi controls (all >255), routing them to `extract_char`. Holds for `Interpol=true` and uppercase-sigil `Interpol=false` alike.
Axes examined: bidi >255 via fast-path bypass attempt, break ≤255 in fast path, char at end-of-string before terminator, Interpol true/false | Not examined: none.
Basis: verified (read interpolation 19–112; traced `"a<VT>"`, `"a<RLO>"` by hand through the clauses).

- **Mixed-script / restricted-codepoint / non-NFC identifier rejection** — `tokenize_identifier/5`, lines 1400–1442; `tokenize_alias`, 1485–1495.
Reason: The `{error,{mixed_script,…}}` and `{error,{unexpected_token,…}}` arms return an error/`{unexpected_token,Length}` marker and never the `{Kind,…}` success shape, so the caller (687–740) cannot accept the token; the confusable-skeleton/NFKC machinery (1444–1475) only builds a *suggestion* message and still returns an error (claim CONFUSABLE_SKELETON_SUGGESTION). Aliases with `not Ascii or Special/=[]` hard-error (ALIAS_ASCII_ONLY).
Axes examined: mixed_script arm, unexpected_token arm, suggestion→no_suggestion fall-through, alias non-ASCII/punctuation | Not examined: the underlying script/NFC/Identifier_Status tables themselves (owned by `String.Tokenizer`, out of this file).
Basis: asserted.
Depends on: `lib/elixir/unicode/tokenizer.ex` (`String.Tokenizer`) — its `tokenize/1` actually returns `{error,{mixed_script,…}}` / `{error,{unexpected_token,…}}` for restricted/mixed/non-NFC identifiers rather than a success tuple (covered by memories [[elixir-tokenizer-uts39-mixed-script-satisfied]] and [[elixir-tokenizer-uts39-restricted-satisfied]]).

- **Unicode escape scalar/well-formedness validation** — delegated to `elixir_interpolation` `unescape_unicode`/`unescape_hex`/`append_codepoint`, lines 221–261.
Reason: `\u{…}` (≤6 hex) and `\uHHHH` decode via `list_to_integer(_,16)` then `<<Codepoint/utf8>>`, whose bit-syntax rejects surrogates and >U+10FFFF (`badarg` → `throw "invalid or reserved Unicode code point"`); malformed `\x`/`\u` throw descriptive errors (claims UNICODE_ESCAPE_SCALAR_ONLY, UNICODE_ESCAPE_WELLFORMED).
Axes examined: surrogate D800, U+110000, U+10FFFF (valid), U+FFFFFF (6×F), truncated `\x`/`\u` | Not examined: none.
Basis: verified — `ran erl`: `<<16#D800/utf8>>`→badarg, `<<16#110000/utf8>>`→badarg, `<<16#10FFFF/utf8>>`→4 bytes, `<<16#FFFFFF/utf8>>`→EXIT.

- **Char literal `?X` accepts raw bidi/break control codepoints** — lines 234–288 (esp. 271).
Reason: `?<RLO>`, `?<VT>`, `?<LS>` etc. are accepted as `{char,_,Codepoint}` (only bare `?\r`/`?\ \r` are special-cased, 230/267). This is *not* one of the three surfaces the Trojan-Source obligation covers (identifiers/strings/comments), and it is not weaponizable: a char literal holds exactly one codepoint, so the matched isolate/pop pair a reordering attack needs cannot be formed, and any lone bidi pop elsewhere in code position errors as `unexpected_token`. Attacker uplift: none beyond a single already-visible `?`-prefixed control char. Accepting any operand is the intended `?X` feature.
Axes examined: all 9 bidi controls, VT/FF/NEL/LS/PS, bare-CR special case | Not examined: multi-char reordering (structurally impossible — one codepoint per literal).
Basis: asserted.
Depends on: no in-file/parser path concatenates char-literal codepoints back into a source-reordering region — char tokens become integer values, not source text.

- **Large number-literal cost** — `tokenize_number`/`tokenize_hex/octal/bin` → `list_to_integer/1,2`, lines 1282–1346.
Reason: Cost is only mildly super-linear on modern OTP (subquadratic bignum conversion) and bounded by source length; not a practical DoS.
Axes examined: decimal & base-16, 100k–800k digits, underscore padding | Not examined: none.
Basis: verified — `ran erl`: decimal 800k digits=66ms, hex 800k digits=15ms (≈N^1.4, not N²).

- **Unbounded recursion / list growth (nested containers, interpolation, Tokens/Terminators/Warnings)** — `tokenize/5`, `check_terminator`, `elixir_interpolation:extract`↔`elixir_tokenizer:tokenize`.
Reason: `tokenize/5` is tail-recursive (container nesting builds O(input) `Terminators`/`Tokens` lists, no C-stack growth); only `#{…}` interpolation is non-tail, and its depth is bounded by source nesting depth ≤ input size — inherent to any parser, no amplification.
Axes examined: container nesting (tail), interpolation nesting (depth≤input), warnings/tokens growth (O(input)) | Not examined: exact heap ceiling under adversarial deep nesting (memory-bound, not a distinct code defect).
Basis: asserted.
Depends on: `elixir_interpolation.erl` — `extract`'s `#{…}` recursion into `elixir_tokenizer:tokenize/4` has no per-level multiplier (verified by read, interpolation 61–79).

- **`lists:nthtail(Length-1, String)` crash** — lines 591, 736.
Reason: `Length` is the length of a token that is a prefix of `String`, so `Length-1 < length(String)`; `nthtail` always leaves ≥1 element — no `function_clause`.
Axes examined: Length=len(String), Length<len(String), empty-tail | Not examined: none.
Basis: verified (read 580–594, 687–740, 745; `Length` originates as `length(Wrong)` with `Wrong ⊆ String`).

- **Global config read** — `elixir_config:identifier_tokenizer()`, line 128.
Reason: Read-only fetch of operator-set compiler config; no external-input write to global state in this file.
Axes examined: read vs write, per-parse locality | Not examined: none.
Basis: verified.

- **`static_atoms_encoder` callback dispatch** — lines 1087–1107.
Reason: A caller-supplied `fun/2` invoked per its documented `{ok,Term}|{error,Binary}` contract; a non-conforming return crashes the caller only and is an internal contract violation (out of scope).
Axes examined: `{ok,_}`, `{error,Binary}` | Not examined: non-tuple return (caller contract, excluded).
Basis: asserted.
Depends on: caller passing a `static_atoms_encoder` honoring the `{ok,_}|{error,binary()}` spec.

- **`format_error/1` → `elixir_utils:characters_to_binary`** — lines 1920–1923.
Reason: Operates on the tokenizer's own error tuples whose tokens are already valid codepoints (source enters as a validated charlist via `elixir:string_to_tokens`); `characters_to_binary` surfaces any encoding error as a `UnicodeConversionError` rather than silent garbage (claim UTF8_DECODE_ERRORS_SURFACED, `elixir_utils.erl:147–155`).
Axes examined: valid-codepoint error tokens, invalid-UTF-8 surfacing behavior | Not examined: whether every caller pre-validates the charlist.
Basis: asserted.
Depends on: `elixir.erl` — `string_to_tokens`/`string_to_quoted` convert the input binary to a charlist (raising on invalid UTF-8) before calling `tokenize/4`.

## Did I verify this?
- Atom-length guard, escape scalar validation, number-literal cost, `system_limit` boundary — **verified** by running Erlang; `Not examined: none`.
- Bidi/break in comments, mixed_script/unexpected_token fall-through, bounded operator atoms, `nthtail`, config read — **verified** by reading the full clause set and tracing; `Not examined: none`.
- Bidi/break in strings/sigils — **verified** by reading `elixir_interpolation.erl` in full and hand-tracing adversarial strings; `Not examined: none`.
- Mixed-script/restricted/NFC **tables** — remains **asserted**; the enforcement wiring in *this* file is verified, but the codepoint classification lives in `String.Tokenizer` (`lib/elixir/unicode/tokenizer.ex`). `Depends on:` recorded; corroborated by memories [[elixir-tokenizer-uts39-mixed-script-satisfied]] and [[elixir-tokenizer-uts39-restricted-satisfied]].
- Char-literal bidi acceptance — **asserted** (non-exploitability is a structural argument: one codepoint per literal); `Depends on:` recorded.
- Interpolation recursion depth, `static_atoms_encoder`, `format_error` charlist provenance — **asserted** with explicit `Depends on:` lines for the cross-file pass (`elixir_interpolation.erl`, caller contract, `elixir.erl`).

## Risks / caveats
- **Atom-table exhaustion is the real residual risk, and it is a caller responsibility, not a defect here.** With the default scope (`existing_atoms_only=false`, no `static_atoms_encoder`), every distinct identifier/atom/alias/multi-letter-sigil in the source interns a new atom. A host that feeds attacker-controlled source to `Code.string_to_quoted/2` (formatters, language servers, doc tooling) can be driven to the ~1M atom limit and crash the BEAM node. `Code.string_to_quoted/2` documents this; mitigation is to pass `:existing_atoms_only` or `:static_atoms_encoder`, both correctly honored by `unsafe_to_atom`.
- **Three Unicode obligations are satisfied only in concert with two sibling modules.** Bidi/break in string & sigil bodies is enforced in `elixir_interpolation.erl:extract_char`; identifier NFC/mixed-script/restricted classification is enforced in `String.Tokenizer` (`lib/elixir/unicode/`). This file enforces bidi/break only for comments and merely *routes* the identifier-security verdicts. If either sibling regresses, this file will not independently catch the bypass.
- **Minor in-file inconsistency (hardening, not a vuln):** char literals `?<control>` accept raw bidi/break controls that are rejected everywhere else; harmless today (single codepoint, no matched reordering possible) but worth a `?bidi`/`?break` guard on the `?`-literal clauses if the maintainers want a uniform "no raw bidi/break in source" rule.
- Toolchain note: the sandbox `erl` self-reported "17.0.2" but exhibits modern OTP semantics (structured `error_info`, subquadratic `list_to_integer`); the primitive results above should be re-confirmed on the exact production OTP if precision matters.

