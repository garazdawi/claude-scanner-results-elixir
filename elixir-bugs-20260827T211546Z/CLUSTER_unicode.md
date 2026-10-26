# Cluster cross-file findings

#### `CSCAN-cluster-bidi-dir-omits-arabic-number-class`

- **Finding ID** — `CSCAN-cluster-bidi-dir-omits-arabic-number-class`
- **Severity** — Medium
- **Confidence** — medium
- **Files involved** —
  - `unicode/tokenizer.ex:63-68` (bidi-class bucketing), `unicode/tokenizer.ex:343-344` (`dir_rtls`/`dir_neutrals` construction), `unicode/tokenizer.ex:350-360` (`dir/1` definition)
  - `unicode/security.ex:126-140` (`bidi_skeleton`, `any_rtl?`), `unicode/security.ex:166-192` (`unbidify`, including the `:weak_number` branch at 171-172)
- **Finding shape** — Inherited assumption broken (file A's `unbidify` rests on "`dir/1` classifies every bidi-weak/neutral identifier codepoint"; file B's `dir/1` does not).
- **Evidence** —
  - `tokenizer.ex:64-68` buckets a codepoint's bidi class into RTL only for `R AL`, and into neutral only for `WS ON CS EN ES ET NSM`. The `true ->` fall-through drops everything else. The **`AN` (Arabic Number)** class is in neither list.
  - Arabic-Indic digits `U+0660..U+0669` are `Nd` (bidi class `AN`) and `Recommended` in `IdentifierType.txt:116`, so they survive `id_continue` and are valid inside identifiers (verified against `UnicodeData.txt:1595` `0660;…;Nd;0;AN;…`). Because `AN ∉ {R,AL,WS,ON,CS,EN,ES,ET,NSM}`, they never enter `dir_rtls`/`dir_neutrals` (`tokenizer.ex:343-344`) and are not `48..57`, so `dir/1` returns `:ltr` for them via the fallback at `tokenizer.ex:360`.
  - `security.ex:170` switches on `String.Tokenizer.dir(head)`. `unbidify` has a dedicated `:weak_number` branch (`security.ex:171-172`) precisely so numbers get pulled toward the surrounding directional span per UAX #9 W2. An Arabic-Indic digit, classified `:ltr`, instead hits the `:ltr when part_dir == :rtl` clause (`security.ex:180-181`), which **terminates the RTL run and reverses it**, treating the digit as strong left-to-right text. The perceived (display) order for AN inside an RTL run is not that of strong LTR text, so the resulting `bidiSkeleton` does not match the visual order.
- **What the per-file pass missed** — A `tokenizer.ex`-only reviewer sees `dir/1` return `:ltr` for an unlisted bidi class and has no signal it is wrong, because `dir/1` has no in-file consumer that cares about `AN`; a `security.ex`-only reviewer sees `unbidify` handle `:weak_number`/`:neutral`/`:rtl`/`:ltr` exhaustively and reasonably assumes its supplier `dir/1` returns the correct UAX #9 class. Only reading both together reveals that the classifier omits a class the reorderer depends on, so RTL identifiers containing Arabic-Indic digits get a mis-ordered bidi skeleton (a bidi-confusable pair can be missed, weakening the Trojan-Source/confusable warning).
- **Suggested fix** — Add `AN` to the weak-number path so `unbidify` sees it as a number rather than strong LTR. Minimal change at `tokenizer.ex:64-68`: give `AN` its own bucket (e.g. a `dir_weaks` list) and emit `:weak_number` for those codepoints in `dir/1`, mirroring the `48..57` clause at `tokenizer.ex:350` — or, at least, route `AN` into `dir_neutrals` so it attaches to the surrounding run instead of terminating it.
