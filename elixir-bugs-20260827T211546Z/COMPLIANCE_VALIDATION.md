# Validator verdicts

Automated prove-or-disprove pass over every `CSCAN-*` finding in the\
sibling per-file reports. Each verdict is one Claude call with both-\
sides steelman prompt at `--effort medium`. Read the **Reasoning**\
line per finding to decide whether to trust the verdict.

## Summary

| Verdict | Count |
|---|---|
| real | 2 |
| fp | 0 |
| uncertain | 0 |
| unknown (validator failed) | 0 |
| **total** | **2** |

## Real findings (2)

<!-- cscan-verdict CSCAN-elixir-rfc-mixed_number_detection real -->

### Validator verdict for CSCAN-elixir-rfc-mixed_number_detection

**Verdict:** real\
**Confidence:** high\
**Files read:** /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/tokenizer.ex, /Users/lukas/code/claude-scanner/elixir/lib/elixir/pages/references/unicode-syntax.md, /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/UnicodeData.txt, /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/IdentifierType.txt

**Steelman (real):** The cited code matches exactly. Line 77 adds General_Category `Nd` codepoints to `continue`, and the restricted filter (lines 116–125) only strips codepoints that are neither `Inclusion` nor `Recommended`. I confirmed against the data files that Devanagari digits `0966..096F` are `Nd` (UnicodeData.txt) and `Recommended` (IdentifierType.txt), so they survive the filter and land in `unicode_continue`, becoming valid identifier continue characters in production tokenization. `validate/2` (lines 483–522) performs only NFC normalization and the mixed-script scriptset-intersection check — grep across `unicode/*.ex` found no digit-base/number-system logic anywhere in code (only data files matched). `codepoint_to_scriptset` maps ASCII digits to `@top` (line 575, script-neutral) and non-ASCII `Nd` digits to their own script's scriptset (lines 578–582). So an identifier like Devanagari-letter + ASCII `0` + Devanagari `०` keeps a non-empty (Devanagari) scriptset and passes, mixing two decimal number systems — precisely what UTS #39 §5.3 (SHOULD) asks implementations to detect. The doc premise at unicode-syntax.md:147 ("Elixir does not support Unicode numbers") is factually false.

**Steelman (FP):** §5.3 is a SHOULD, and mixed-number detection is a spoofing-resistance nicety, not a correctness requirement — one could argue the project made a deliberate, documented non-goal (C5 declared "inapplicable" at unicode-syntax.md:143–147), making this an intended design decision rather than a deviation. Additionally the exploit is narrow and contrived (requires deliberately combining ASCII and non-ASCII digits within a single script's identifier).

**Reasoning:** The rubric explicitly permits real standards deviations whose only consequence is a non-canonical/interop/spoofing correctness issue, at Low severity — security leverage is not required and SHOULD-level clauses count. The FP steelman's strongest card is the documented C5 non-goal, but the rubric's "intended by design" exception requires the stated premise to hold; here the premise ("Elixir does not support Unicode numbers") is demonstrably false — I verified Devanagari `Nd` digits are `Recommended` and reach `unicode_continue`, so they ARE accepted. A non-goal resting on an incorrect factual premise is an omission owed, not an architectural boundary. The code is the production tokenizer (no test-only flag gates this path), the deviation is reachable, and the check genuinely does not exist. That tips the balance to `real`, Low severity as the finding states.

<!-- cscan-verdict CSCAN-elixir-rfc-implicit_level_resolution real -->

### Validator verdict for CSCAN-elixir-rfc-implicit_level_resolution

**Verdict:** real\
**Confidence:** high\
**Files read:** /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/security.ex, /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/tokenizer.ex

**Steelman (real):** The cited code exists exactly as described: `security.ex:171-172` handles `:weak_number` by `{[], part_dir, [head] ++ neutrals ++ part, acc}`, prepending each digit individually to the current part just like a strong RTL char (the `^part_dir` clause at 177-178). Since an RTL part is built by prepending and never internally re-reversed (see the `:rtl` finalization at line 190, `Enum.reverse(acc, neutrals ++ last_part)`), consecutive digits in an RTL run come out reversed. `String.Tokenizer.dir(i) when i in 48..57` returns `:weak_number` for digits 0-9 (tokenizer.ex:350). This is live production code: `confusable_skeleton/1` (106) → `bidi_skeleton/1` (126) → `unbidify/1` (166) is the Elixir compiler's identifier-confusability check, run for any identifier containing RTL chars. I verified by execution: `S.unbidify([Aleph, ?1, ?2])` returned `[50, 49, 1488]` (= "2","1",Aleph) and `S.unbidify([Aleph, ?1, ?2, Bet])` returned `[1489, 50, 49, 1488]` (= Bet,"2","1",Aleph). The UAX #9 correct visual order keeps EN digits LTR (`[49,50,1488]` = "1","2",Aleph), because European numbers within an RTL run sit at an even embedding level and are double-reversed. The code single-reverses them — a genuine, reachable deviation.

**Steelman (FP):** The finding cites I1/I2 (whose literal subject is *NIs*, neutral characters), whereas the digit-ordering behaviour is actually governed by W2/W7 (EN classification) and L1/L2 (implicit reordering levels) — one could argue the exact clause reference is imprecise. One could also argue the function is a deliberately "drastically simplified" skeleton (comments 162-166 cite UTS39-28 §4 permitting simplification when no explicit directional-formatting chars are present), so perfect UAX #9 fidelity is not contractually promised, and the impact is confined to soft confusability *warnings* rather than any hard rejection or security boundary.

**Reasoning:** The behavioural claim is settled by execution: `unbidify` returns digits in reversed order within an RTL run (`[50,49,1488]` where correct is `[49,50,1488]`), on a path that runs in the production Elixir compiler. The FP steelman does not discharge this: the rubric explicitly says a real, reachable standards deviation is `real` even when its only consequence is non-canonical output or an interop/lint mismatch, and even when the clause citation is slightly off-target the underlying normative requirement (numbers embedded in RTL runs must not be visually reversed with the surrounding strong characters) is genuinely violated. The "simplified algorithm" defense fails because the simplification permitted by UTS39-28 §4 concerns dropping the explicit-formatting stack, not licensing incorrect number ordering — the code's own comments (159-160) state it attaches "weak number types according to uax9," which it then does incorrectly. Low severity is appropriate, but it is a real conformance gap.

