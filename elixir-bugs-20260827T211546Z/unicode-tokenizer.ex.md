# /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/tokenizer.ex
<!-- sha256: 3612eb4fd3a00389146aa5018695eaa2ad171cffb8abb179f44bcc4776ff4f15 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T00:13:54Z_

I have a verified finding. Writing the report.

# Bug-hunt report — `lib/elixir/unicode/tokenizer.ex`

## Summary
The public `dir/1` bidi-classifier misclassifies non-ASCII decimal-digit identifier characters (notably Arabic-Indic digits, bidi class `AN`) as `:ltr` instead of `:weak_number`, causing the bidi-confusable skeleton to be computed in the wrong visual order for RTL identifiers that contain such digits.

## Bugfixes

#### `CSCAN-tokenizer-dir-nonascii-digit-weak`
- **Finding ID** — `CSCAN-tokenizer-dir-nonascii-digit-weak`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `lib/elixir/unicode/tokenizer.ex:350` and `:360` (`dir/1`); root inputs at `:65-66` (bidi whitelist), `:344` (neutrals filter), comment `:348-349`.
- **Issue** — `dir/1` treats only ASCII `48..57` as `:weak_number` (line 350). Its generated clauses cover `:rtl` (bidi `R`/`AL`, line 65) and `:neutral` (bidi `WS ON CS EN ES ET NSM`, line 66); every other codepoint falls through to `def dir(i) when is_integer(i), do: :ltr` (line 360). Arabic-Indic digits `U+0660..U+0669` have bidi class **`AN`** (Arabic Number), which appears in neither whitelist, so they are classified `:ltr`. But those digits are category `Nd` and are valid identifier-continue characters, so they occur in real identifiers. The comment at lines 348–349 ("the only weaks allowed in idents are numbers 0..9") states an assumption that is factually false. (Extended Arabic-Indic digits `U+06F0..U+06F9`, bidi class `EN`, are likewise classified `:neutral` rather than `:weak_number`.)
- **Class** — Missing clause / non-total classification (semantic misclassification), with a Documentation-mismatch component (the line 348–349 comment).
- **Impact** — In the consumer `String.Tokenizer.Security.unbidify/1`, a `:weak_number` stays inside the current directional run, whereas an `:ltr` character *breaks* an RTL run. So an Arabic-Indic digit inside an Arabic identifier prevents the run from being reversed, and the resulting **bidi skeleton is built in byte order instead of visual (reversed) order**. Since `bidi_skeleton/1` is the Trojan-Source / homoglyph confusable-detection mitigation (UTS #39 §4 / UAX #9), identifiers mixing Arabic letters with Arabic-Indic digits get a wrong skeleton, so confusable pairs involving them can silently escape the whole-file confusability warning (false negative), or hash into the wrong group. It is advisory (a warning, not a hard error) and needs a specific script+digit combination, hence Medium.
- **Evidence** — `verified`.
  - `String.Tokenizer.dir(0x0669)` ⇒ `:ltr`; `dir(?0)` ⇒ `:weak_number`; `dir(0x06F9)` ⇒ `:neutral`.
  - Reachable identifier: `String.Tokenizer.tokenize([0x0628,0x0660,0x0669,0x0627])` ⇒ `{:identifier, [1576,1632,1641,1575], [], 4, false, []}`.
  - Divergent public skeleton:\
    `String.Tokenizer.Security.bidi_skeleton([0x0627, 0x0669, 0x0628])` ⇒ `[~c"l", "\x{669}", "\x{628}"]` (unreversed), vs\
    `String.Tokenizer.Security.bidi_skeleton([0x0627, ?0, 0x0628])` ⇒ `["\x{628}", "0", ~c"l"]` (correctly reversed).\
    `unbidify([0x0627, 0x0669, 0x0628])` ⇒ `[1575,1641,1576]` (run not reversed) vs `unbidify([0x0627, ?0, 0x0628])` ⇒ `[1576,48,1575]`.
- **Suggested fix** — Classify all bidi weak-number identifier characters as `:weak_number`, not just ASCII. Simplest: in the `dir/1` generation, add `AN` (and `EN`) code points to a weak-number set (mirroring how `48..57` is special-cased) rather than letting them fall into the neutral/`:ltr` buckets — e.g. extend line 66's handling so bidi `AN`/`EN` identifier code points emit `:weak_number` clauses, and drop the ASCII-only assumption in the line 348–349 comment.

## Risks / caveats
- Scope: only `tokenize/1` and `dir/1` are public (`def`) runtime entry points; the rest of the file is compile-time table generation. I focused bug-hunting on the two runtime functions and the generation logic that determines which code points they accept/classify.
- I did not read/audit `unbidify` for defects (it lives in `security.ex`, out of scope); I only used it to establish the *impact* of the `dir/1` misclassification. Its `case` was confirmed total for the four values `dir/1` can return, so the misclassification degrades correctness rather than crashing.
- The severity assumes the confusability check is advisory (a warning). If a downstream configuration promotes it to a hard error, the same defect becomes a correctness/availability issue of correspondingly higher severity.

### Ruled out
- **`tokenize/1` / `dir/1` crashing on hostile input** — fuzzed 200k random code-point lists (mixing ASCII, Latin-1, BMP, and full `U+0..U+10FFFF`, surrogates removed) through both functions; no `FunctionClauseError`/`throw`/`exit`. Basis: `verified`.
- **UnicodeData First/Last range reconstruction (lines 49–61)** — the First row emits `[first]` and stashes `first+1`; the Last row emits `last..first+1//-1`; union covers `first..last` with no gap/overlap, and both rows carry identical category/bidi. Basis: `verified` (module compiles and CJK-range code points like `28450` tokenize with correct Han scriptset).
- **µ (U+00B5) custom normalization giving scriptset `@top` (lines 256–261)** — `union(top, Greek)` collapsing to `@top` for `U+03BC` is intentional per the lines 8–14 comment (a Common member makes the pair usable anywhere); `normalize_start(0x00B5)` correctly routes through the NFKC path. Basis: `verified` (`tokenize([0x00B5, ?x])` ⇒ `{:identifier, [956,120], …, [:nfkc]}`).
- **`invalid_prefix = different? && invalid_normalized(...)` treating `[]` as truthy (line 484)** — `invalid_normalized` returns `false` or a non-empty original-prefix charlist; `original_acc` is always non-empty on this path, so an empty-list truthiness trap is unreachable. Basis: `asserted` (code-path reasoning; the `false` short-circuit was also observed to suppress the clause in normalizing inputs like `[?e,0x0301]`).
- **`rangify` merging code points with differing scriptsets** — the first reduce clause binds the element and accumulator scriptset to the same variable plus `number == first-1`, so only identical-scriptset consecutive code points merge; differing ones start a new range. Basis: `verified` (augmented CJK and mixed-script code points return correct masks in live `tokenize` runs).
- **Mixed-script breakdown `Map.fetch!(@indexed_scriptsets, …)` raising (line 501)** — every set bit in a code point's mask indexes a name present in `@indexed_scriptsets`; exercised the multi-bit script-extension path (`[?a, 0x0669]`) with no crash. Basis: `verified`.

