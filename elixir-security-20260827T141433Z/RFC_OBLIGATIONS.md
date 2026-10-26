<!-- rfc-governed: yes -->
<!-- standards: Unicode UAX #31; Unicode UTS #39; Unicode UTS #55; Unicode UAX #24; Unicode UAX #15; Unicode UAX #9 -->

# Security-obligation checklist — `elixir/lib/elixir/src`

This folder is the **Elixir compiler front-end written in Erlang**: tokenizer, interpolation/escape handling, parser (`elixir_parser.yrl`), macro expander, and BEAM code generation. Most of it is OTP-/language-internal, but the **tokenizer and interpolation modules implement Unicode source-text handling** — identifier syntax, identifier security (mixed-script/confusables), NFC normalization requirements, unsupported-newline handling, bidirectional-control rejection (Trojan Source), and Unicode escape validation. Those are governed by external Unicode standards with normative, security-bearing clauses that a single-file reviewer could not derive from the code alone, so the RFC/compliance verdict is `yes`.

The heavy identifier-security *tables and algorithms* (`String.Tokenizer`, `String.Tokenizer.Security`, driven by the Unicode Character Database) live in the sibling `lib/elixir/unicode/` directory, **not** in this folder; this folder is the caller/enforcer. Sub-claims below are scoped to what this folder itself owns and enforces.

## Applicable standards / contracts

### External standards (compliance pass adjudicates ONLY these)
- **Unicode UAX #31 — Identifier and Pattern Syntax** — constrains which codepoints may start/continue an unquoted identifier and requires normalization; enforced here via `tokenize_identifier` delegating to `identifier_tokenizer` and rejecting non-conforming forms.
- **Unicode UTS #39 — Unicode Security Mechanisms** — mixed-script/confusable identifier detection and the confusable "skeleton" algorithm; this folder consumes `{error,{mixed_script,…}}` and `confusable_skeleton/1` to reject/suggest.
- **Unicode UTS #55 — Unicode Source Code Handling** — explicitly cited in `elixir_tokenizer.hrl:38`; governs which whitespace/newline codepoints a compiler must treat as line breaks and the NFC requirement for identifiers.
- **Unicode UAX #24 — Script Property** — the Script values underlying the mixed-script check.
- **Unicode UAX #15 — Normalization Forms (NFC/NFKC)** — identifiers must be in NFC; `suggest_simpler_unexpected_token_in_error` uses `unicode:characters_to_nfkc_list/1` and NFC messaging.
- **Unicode UAX #9 — Bidirectional Algorithm / Trojan Source (CVE-2021-42574)** — the `?bidi/1` control-codepoint set in `elixir_tokenizer.hrl:27-35` (sourced from trojansource.codes) that must be rejected in source text.

### Mechanically-cited standards that do NOT govern this folder
- **RFC 3986 (URI, 13×)** — Elixir `URI` module (`lib/elixir/lib/uri.ex`), not the compiler src.
- **RFC 3339 (1×)** — `Calendar`/`DateTime`, not this folder.
- **RFC 4648 (1×)** — `Base` module, not this folder.
- **RFC 8259 (1×)** — `JSON` module, not this folder.
- **UTS #18 (1×)** — `Regex` module, not this folder.
- **UAX #44 + UCD data files** — define the format of `unicode/*.txt`, consumed by the `unicode/` generator that produces `String.Tokenizer*`; not implemented in this src folder (it is the downstream *caller*).

### OTP-internal contracts (per-file context, NOT compliance obligations)
- `-spec` return shapes across `elixir_errors.erl`, `elixir_erl.erl`, etc. (e.g. `no_return()` on `compile_error`/`erl_warn`) — internal contracts only.
- Tokenizer return tuples `{Kind,Acc,Rest,Length,Ascii,Special}` and the `#elixir_tokenizer{}` record — internal to the tokenize protocol, not external.
- `elixir_sup`/`elixir_code_server` OTP supervision/gen_server conventions.

## Checkable sub-claims

1. **BIDI_CONTROL_REJECTED** — The nine bidirectional formatting controls (LRE 202A, RLE 202B, LRO 202D, RLO 202E, LRI 2066, RLI 2067, FSI 2068, PDF 202C, PDI 2069) must be rejected in source text (identifiers, strings, comments), not silently passed through. (`?bidi/1`, UAX #9 / Trojan Source; `elixir_tokenizer.hrl:27`, `elixir_tokenizer.erl:1357`.)
2. **UNSUPPORTED_NEWLINES_REJECTED** — VT (000B), FF (000C), bare CR (000D not followed by LF), NEL (0085), LS (2028), PS (2029) must be handled/rejected per the `?break` set, and CRLF must be handled explicitly before `?break`. (UTS #55; `elixir_tokenizer.hrl:41`.)
3. **IDENTIFIER_NFC_REQUIRED** — Unquoted Unicode atoms/variables/calls must be in NFC form; a non-NFC identifier must error (with an NFC suggestion), not be accepted or auto-normalized silently. (UTS #55 / UAX #15; `elixir_tokenizer.erl:1462`.)
4. **MIXED_SCRIPT_REJECTED** — Mixed-script confusable identifiers must be rejected (the `{error,{mixed_script,…}}` path must lead to a compile error, never fall through to acceptance). (UTS #39 §5; `elixir_tokenizer.erl:1414`.)
5. **RESTRICTED_CODEPOINTS_ONLY** — Only allowed ID_Start/ID_Continue codepoints may form identifiers; unexpected codepoints must produce `unexpected_token`, not be tokenized. (UAX #31 / UTS #39; `elixir_tokenizer.erl:1426`.)
6. **ALIAS_ASCII_ONLY** — Module aliases must be restricted to ASCII letters without punctuation; non-ASCII/`Special` aliases must error. (Elixir alias rule atop UAX #31; `elixir_tokenizer.erl:1485-1489`.)
7. **UNICODE_ESCAPE_SCALAR_ONLY** — `\uHHHH` / `\u{H*}` escapes that decode to surrogates (D800–DFFF) or beyond U+10FFFF must be rejected ("invalid or reserved Unicode code point"), not encoded. (Unicode scalar-value rule; `elixir_interpolation.erl:254-261`.)
8. **UNICODE_ESCAPE_WELLFORMED** — Malformed `\u`/`\x` escapes must throw a descriptive error rather than partial/undefined decoding. (`elixir_interpolation.erl:225,251`.)
9. **CONFUSABLE_SKELETON_SUGGESTION** — When a token fails, the confusable-skeleton (UTS #39) and NFKC forms are used only to *suggest* a safe alternative — the original confusable/non-NFC token must still be rejected, not accepted because a skeleton matched. (`elixir_tokenizer.erl:1444-1467`.)
10. **BIDI_COMMENT_NO_BYPASS** — Bidi rejection in comments must not be bypassable by comment context; a bidi char in a comment must error like elsewhere. (`elixir_tokenizer.erl:1357`.)
11. **UTF8_DECODE_ERRORS_SURFACED** — Invalid UTF-8 in source/IO conversion must raise a conversion error, not yield replacement/garbage silently. (`elixir_utils.erl:139-155`, `unicode:characters_to_*`.)
12. **GRAPHEME_CLUSTER_SEGMENTATION** — Character advancement in interpolation uses `unicode_util:gc/1` (grapheme clusters), so column/position accounting stays consistent with Unicode text segmentation. (`elixir_interpolation.erl:94`.)

## Per-file checking instructions

For each sub-claim that could plausibly apply to the file under review, search for the code that satisfies it. The high-value files are `elixir_tokenizer.erl`, `elixir_tokenizer.hrl`, `elixir_interpolation.erl`, and `elixir_utils.erl`; most other files in this folder (parser, expander, code-gen, config, supervision) will not touch these Unicode obligations — say so briefly rather than forcing a match. If a file declares the check or default that satisfies a claim (e.g. the `?bidi`/`?break` macros, a `mixed_script`/`unexpected_token` rejection branch, an NFC/scalar-value guard), name the specific clause and treat the claim as satisfied. If a Unicode-handling code path exists where you would expect one of these checks to be enforced but the satisfying code is absent or the error branch falls through to acceptance, report it as a gap under the appropriate failure-mode class (identifier-security bypass, Trojan-Source/bidi injection, malformed-escape/UTF-8 handling). Do **not** raise findings from the OTP-internal contracts section — those are per-file context only.