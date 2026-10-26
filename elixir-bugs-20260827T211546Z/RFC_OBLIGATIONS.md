<!-- rfc-governed: yes -->
<!-- standards: Unicode UAX #31; Unicode UTS #39; Unicode UAX #24; Unicode UAX #44; Unicode UAX #15; Unicode UAX #9 -->

# Documented-contract / behavioural-obligation checklist — `elixir/lib/elixir`

This folder is the **Elixir compiler front-end** (Erlang sources: tokenizer, parser, expander, code generation) plus, in `unicode/`, the **Unicode data generators** (`tokenizer.ex`, `security.ex`, `unicode.ex`) that emit the `String.Tokenizer` / `String.Tokenizer.Security` modules the tokenizer calls at compile time. The presence of the **Unicode Character Database** tables (`UnicodeData.txt`, `Scripts.txt`, `ScriptExtensions.txt`, `PropList.txt`, `PropertyValueAliases.txt`, `IdentifierType.txt`, `SpecialCasing.txt`, `confusables.txt`) beside those generators is decisive: this folder *implements* the Unicode identifier and identifier-security algorithms. `elixir_tokenizer.erl` enforces the results (mixed-script rejection, NFC form, confusable-skeleton suggestions) at tokenization time (`src/elixir_tokenizer.erl:1400-1483`), and the repo's own conformance page (`pages/references/unicode-syntax.md`) states verbatim that Elixir conforms to **UAX #31 v17.0** and **UTS #39 v17.0**. This is a standardized-algorithm conformance surface a single-file reviewer cannot derive from the code alone — hence `rfc-governed: yes`.

## Applicable standards / contracts

### External standards (compliance pass adjudicates ONLY these)

- **Unicode UAX #31 (Unicode Identifier and Pattern Syntax)** — defines the `ID_Start`/`ID_Continue` code-point sets (from General Categories + `Other_ID_Start`/`Other_ID_Continue`, minus `Pattern_Syntax`/`Pattern_White_Space`) and the normalized-identifier requirement (R4). Governs `unicode/tokenizer.ex` set construction and `elixir_tokenizer.erl` identifier tokenization.
- **Unicode UTS #39 (Unicode Security Mechanisms)** — General Security Profile (§C1: reject `Identifier_Status=Restricted`), mixed-script detection via scriptsets (§5.1), the confusable-skeleton algorithm (§4: NFD → prototype substitution → NFD) and bidi-skeleton. Governs `unicode/security.ex`, the scriptset machinery in `unicode/tokenizer.ex`, and the `mixed_script`/confusable paths in `elixir_tokenizer.erl`.
- **Unicode UAX #24 (Unicode Script Property)** — `Scripts.txt`, `ScriptExtensions.txt`, and short/long alias resolution via `PropertyValueAliases.txt` feed the per-codepoint scriptsets used for mixed-script checks (`unicode/tokenizer.ex:143-186`).
- **Unicode UAX #44 (Unicode Character Database)** — the field layout of `UnicodeData.txt` (codepoint;name;category;…;bidi), `PropList.txt`, `IdentifierType.txt`, and range-first/last `<…, First>`/`<…, Last>` conventions the generators parse (`unicode/tokenizer.ex:42-83`). Also governs `SpecialCasing.txt` case-mapping data.
- **Unicode UAX #15 (Normalization Forms)** — NFC (identifier canonicalization, R4), NFD (confusable skeleton), and NFKC (error-suggestion heuristic) as applied via `:unicode.characters_to_nfc/nfd/nfkc_list` (`elixir_tokenizer.erl:1446`, `unicode/security.ex:106-114`).
- **Unicode UAX #9 (Bidirectional Algorithm)** — the simplified reordering used to build the *bidi* skeleton for RTL-containing identifiers (`unicode/security.ex:126-192`, citing W2 and the UTS #39-28 fast path).

### Standards accounted for but NOT governing this folder

- **RFC 3986 (URI, 13 mentions)** — implemented in `lib/uri.ex` (the `URI` library module), a different component; no bearing on the `.erl` compiler sources or the unicode generators.
- **RFC 3339 (date/time, 1)** — `lib/calendar/naive_datetime.ex`; separate library component.
- **RFC 4648 (base16/32/64, 1)** — `lib/base.ex`; separate library component.
- **RFC 8259 (JSON, 1)** — `lib/json.ex`; separate library component.
- **UTS #18 (regex, 1)** — passing reference in the `Regex`/docs area; not implemented by this folder's tokenizer or generators.
- **UTS #55 (Source Code Handling, 1)** — conceptually adjacent to the tokenizer (source-form handling), but appears only as a documentary reference; the concrete normative obligations enforced here are those of UAX #31 and UTS #39, so no independent clauses are enumerated for it.
- **SpecialCasing.txt** — Unicode case-mapping data; its format falls under UAX #44 above.

### OTP-internal contracts (per-file context, NOT compliance obligations)

- `elixir_tokenizer:tokenize/*` and `String.Tokenizer.tokenize/1` return either `{Kind, Acc, Rest, Length, Ascii, Special}` or `{error, Reason}` tuples — the `.erl` callers pattern-match every documented shape (`elixir_tokenizer.erl:1401-1442`). A silent shape change or unmatched clause would `badmatch` in the caller.
- Tokenizer/parser entry points are documented to return `{error, Reason}` (with location + message) on malformed input rather than raising; the parser (`elixir_parser.yrl`) and `elixir_errors.erl` are the error-formatting contract.
- These are guidance for the per-file reviewer only; the compliance pass MUST NOT treat them as standard-conformance obligations.

## Checkable sub-claims

1. **ID_START_SET_FROM_UCD** — `<Start>` code points must be exactly `[\p{L}\p{Nl}\p{Other_ID_Start}] − \p{Pattern_Syntax} − \p{Pattern_White_Space}` (variables additionally exclude `Lu`/`Lt`; add `_`); derived from `UnicodeData.txt` + `PropList.txt`, not an ad-hoc range. (UAX #31 R1; `unicode/tokenizer.ex:70-108`, `unicode-syntax.md` R1.)
2. **ID_CONTINUE_SET_FROM_UCD** — `<Continue>` must be `ID_Start ∪ {Mn,Mc,Nd,Pc} ∪ Other_ID_Continue` minus the pattern sets; nothing wider. (UAX #31 R1.)
3. **RESTRICTED_CODEPOINTS_EXCLUDED** — Code points with `Identifier_Status=Restricted` (from `IdentifierType.txt`, keeping only `Inclusion`/`Recommended`) MUST NOT be tokenizable as identifiers, except the explicitly listed additional normalizations. (UTS #39 §C1 General Security Profile; `unicode/tokenizer.ex:110-129`.)
4. **IDENTIFIER_NFC_NORMALIZED** — Unquoted atoms/variables/calls MUST be normalized to (or required to be in) NFC form before becoming a token; non-NFC input is rejected or rewritten, never accepted verbatim. (UAX #31 R4 / UAX #15; `elixir_tokenizer.erl:1462`, `unicode-syntax.md` R4.)
5. **MIXED_SCRIPT_REJECTED** — A single identifier whose characters have an empty scriptset intersection (per UAX #24 scripts + script-extensions, with Common/Inherited as `top`) MUST be rejected as `mixed_script`. (UTS #39 §5.1; `elixir_tokenizer.erl:1414`, `unicode/tokenizer.ex:246-279`.)
6. **SCRIPT_AUGMENTATION_APPLIED** — Han/Hiragana/Katakana/Hangul/Bopomofo scriptsets MUST be augmented (Han⇒{Japanese,Korean,Han+Bopomofo}, etc.) so legitimate CJK mixing is allowed while unrelated mixing is not. (UTS #39 §5.1 augmented-script sets; `unicode/tokenizer.ex:220-241`.)
7. **CONFUSABLE_SKELETON_ALGORITHM** — The skeleton MUST be `NFD → per-character prototype substitution (from confusables.txt) → NFD`, matching UTS #39 §4 exactly; no shortcut that skips either NFD pass. (UTS #39 §4; `unicode/security.ex:106-114`.)
8. **CONFUSABLE_WHOLE_FILE_WARNING** — Two identifiers in a file that share a confusable skeleton but differ in code points MUST produce a confusability warning (naming the earlier occurrence). (UTS #39 §4/§5; `unicode/security.ex:22-64`.)
9. **BIDI_SKELETON_FOR_RTL** — Identifiers containing bidi-class R/AL characters MUST be reordered (via the UAX #9-based `unbidify`) before confusable comparison; the pure-LTR fast path (`bidiSkeleton = skeleton`) applies only when no R/AL characters are present. (UTS #39-28 §4 / UAX #9 W2; `unicode/security.ex:126-192`.)
10. **BIDI_CONTROLS_DISALLOWED** — Bidirectional control characters and ZWJ/ZWNJ MUST NOT be permitted inside identifiers (Elixir deliberately does not implement R1a and forbids bidi controls). (UAX #31 R1a note; `unicode-syntax.md:75`.)
11. **ALIAS_ASCII_ONLY** — Aliases (module names) MUST be restricted to ASCII, uppercase-initial, no punctuation; a non-ASCII or punctuated alias is an error. (Elixir profile of UAX #31; `elixir_tokenizer.erl:1485-1489`.)
12. **PATTERN_SYNTAX_WHITESPACE_HONORED** — `Pattern_Syntax` and `Pattern_White_Space` code points (from `PropList.txt`) MUST be subtracted from identifier sets so syntax characters can never appear in an identifier. (UAX #31 R3-restricted; `unicode/tokenizer.ex:87-108,127-129`.)
13. **SCRIPT_ALIAS_RESOLUTION** — Short script tags from `ScriptExtensions.txt` MUST be resolved to canonical names via the `sc` mappings in `PropertyValueAliases.txt` before union into scriptsets; unmapped tags must not silently collapse scripts. (UAX #24; `unicode/tokenizer.ex:151-185`.)
14. **UCD_RANGE_PAIRS_PARSED** — `UnicodeData.txt` `<…, First>`/`<…, Last>` range rows MUST be expanded to the full inclusive code-point range, not treated as two isolated code points. (UAX #44 §4.2.3; `unicode/tokenizer.ex:49-61`.)
15. **NORMALIZED_ERROR_SUGGESTION_SAFE** — When suggesting a simpler form for a rejected token (NFKC / confusable-skeleton retry), the suggestion path MUST NOT itself accept a form that fails identifier tokenization; only a form that re-tokenizes cleanly may be offered. (UAX #15 / UTS #39; `elixir_tokenizer.erl:1444-1468`.)
16. **DOCUMENTED_TOKENIZER_RETURN_SHAPE** — `tokenize/1` results must match one of the documented tuples/`{error,Reason}` shapes so `.erl` callers never `badmatch`. (OTP-internal `-spec`/`@doc`; context only.)

## Per-file checking instructions

For each sub-claim that could plausibly apply to the file in hand, search for code that satisfies it. The `.erl` files (chiefly `elixir_tokenizer.erl`) are where the Unicode obligations are *enforced at tokenization time* — look there for mixed-script rejection (claim 5), NFC handling (claim 4), confusable/skeleton suggestion (claims 7, 15), alias-ASCII restriction (claim 11), and correct matching of the documented tokenizer return shapes (claim 16). The `unicode/*.ex` generators plus the UCD `.txt` tables are where the *sets and algorithms are defined* — check claims 1–3, 6, 9, 12–14 against the generator logic and the data files it reads. If a file declares a set, table parse, augmentation rule, or check that satisfies a claim, name it and the data file it derives from. If a file is one where you'd expect a claim enforced (e.g. an identifier-set generator that omits the `Restricted` subtraction, a skeleton that drops an NFD pass, or a scriptset check that ignores augmentation) and the satisfying code is absent, report a gap as a finding under the appropriate Unicode-conformance / identifier-security failure class, citing the specific UAX/UTS clause above. Ignore the RFC 3986/3339/4648/8259 and UTS #18/#55 references — they belong to other components and are out of scope for this folder.