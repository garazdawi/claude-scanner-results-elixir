# /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/unicode.ex
<!-- sha256: 06390a37b73b005dc3afb442863a5312bec0c133c46104574caf590e4c69c461 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/unicode.ex
<!-- sha256: 06390a37b73b005dc3afb442863a5312bec0c133c46104574caf590e4c69c461 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (hedge-resolution pass; every axis below was executed on the in-repo Erlang/OTP 29 + Elixir 1.21.0-dev toolchain, `ELIXIR_ERL_OPTIONS="+fnu"`)_

## Summary

Re-review of the Elixir Unicode case-mapping / whitespace-breaking generator module (`String.Unicode` and `String.Break`) confirms **no genuine, reachable runtime bugs**. The file on disk matches the reviewed sha256 (`06390a3…c69c461`) exactly. The first-pass reviewer flagged the phrase "did not exhaustively" over the case-mapping-table transcription hedge and the dead-binding hedge. Both hedges are now resolved with **executed** evidence, not asserted invariant: I ran a full diff of `UnicodeData.txt` against the runtime `String.upcase/downcase(_, :default)`, and I grepped the whole tree for consumers of the `decompositions`/`combining_classes` maps and `String.Break.decompose/2`, reading the neighbour modules `unicode/security.ex`, `unicode/tokenizer.ex`, and `lib/string.ex` to confirm normalization routes through OTP, not this module. No findings. No findings dropped (first pass had none). Every ruled-out entry that claims `Not examined: none` was cleared by pushing the relevant axis through the real implementation, cited below.

## Bugfixes

No findings.

## Ruled out

- **`downcase/upcase` crash or mis-handling on invalid/partial UTF-8** — generated multibyte clauses fall through to byte-at-a-time catch-alls (`downcase` line 334, `upcase` line 452) plus `""` terminators (lines 342, 460), so any binary is total. Basis: `verified`. Axes examined: empty binary, lone continuation byte, truncated lead byte, invalid two-byte sequence, ASCII-around-garbage. Not examined: none. **Ran:** `String.downcase(<<>>)`→`""`; `<<0x80>>`→`[128]`; `<<0xC2>>`→`[194]`; `<<0xFF,0xFE>>`→`[255,254]`; `<<0x41,0x80,0x5A>>`→`[97,128,122]`; `String.upcase(<<0x61,0x80,0x7A>>)`→`[65,128,90]`. All returned without error, touching only ASCII bytes.

- **Turkic İ/ı and dotted-I handling (lines 290–303, 424–427)** — clause ordering (two-char `@letter_I,@combining_dot_above` before single `@letter_I`) and mode branching are correct. Basis: `verified`. Axes examined: default vs turkic mode, precomposed U+0130 vs decomposed I+U+0307, round-trip up/down. Not examined: none. **Ran:** `downcase(U+0130,:default)`→`[105,204,135]` (i + U+0307); `downcase(U+0130,:turkic)`→`i`; `upcase("i",:turkic)`→`[196,176]` (U+0130); `upcase("i",:default)`→`I`; `downcase(<<0x49,0xCC,0x87>>,:turkic)`→`i`; `downcase(<<0x49,0xCC,0x87>>,:default)`→`[105,204,135]`. This U+0130→`i`+U+0307 is also the single default-mode downcase divergence in the full-table diff below — the documented SpecialCasing full mapping, not a bug.

- **Greek final-sigma logic in `greek_final_sigma`/`downcase_greek` (lines 276–368)** — accumulator prepend order `[downcased, before | acc]` and the `cased_before`/`cased_letter_binary?` context resolve correctly. Basis: `verified`. Axes examined: medial vs final sigma position, default vs greek mode, sigma between multibyte codepoints. Not examined: none. **Ran:** `downcase("ΣΟΦΟΣ",:greek)`→`[963,959,966,959,962]` (trailing final sigma 0x03C2, leading medial 0x03C3); `downcase("ΣΟΦΟΣ",:default)`→`[963,959,966,959,963]` (no final-sigma rule in default mode); `downcase("λΣλ")`→`[955,963,955]` (medial sigma → 0x03C3, no straddling mismatch).

- **`:binary.match(string, @letter_sigma)` false-positive across codepoint boundaries** — sigma lead byte `0xCE` cannot appear as a UTF-8 continuation byte, so no straddling match; same for the 2-byte-whitespace `do_trim_trailing_lookahead` pattern whose second byte `0xC2` cannot be a continuation byte. Basis: `verified` (UTF-8 byte-class structural fact, corroborated by the `λΣλ` and mixed-whitespace runs). Axes examined: sigma between multibyte codepoints; 2-byte-whitespace lookahead over multibyte tails. Not examined: none — `downcase("λΣλ")` yields the correct medial sigma with no boundary false-positive, and `trim_trailing("abc"<>NBSP<>LS)` (below) correctly stops at the ASCII boundary.

- **`String.Break.trim_trailing` multibyte look-behind (lines 504–554), incl. `size < @whitespace_max_size` short path** — Basis: `verified`. Axes examined: pure-whitespace, mixed 1/2/3-byte whitespace, string shorter than max whitespace size, lone continuation byte, short non-whitespace. Not examined: none. **Ran:** `trim_trailing("abc"<>NBSP<>LS)`→`"abc"`; `trim_trailing(NBSP)`→`""`; `trim(EM_SPACE<>"A"<>EM_SPACE)`→`"A"`; `trim_trailing(<<0x80>>)`→`[128]` (unchanged, no crash); `trim_trailing("ab")`→`"ab"`.

- **`split/1` non-breaking exclusion (line 559, `whitespace -- non_breakable`)** — NBSP trimmed but not split. Basis: `verified`. Axes examined: breaking vs non-breaking whitespace separator. Not examined: none. **Ran:** `split("a"<>NBSP<>"b")`→`["a\u00A0b"]`; `split("a"<>LS<>"b")`→`["a","b"]`.

- **UCD case-mapping parsing obligations relevant to this file** — canonical-vs-compatibility distinction (`<<h,_>> when h != ?<`, line 160) keeps only canonical mappings; `Canonical_Combining_Class` parsed numerically (line 173); `First/Last` and `..` ranges expanded (`expand_range`, `rangify`); SpecialCasing unconditional mappings keystore-merged with UnicodeData simple mappings (lines 201–221). Basis: `verified`. Axes examined: whole `UnicodeData.txt` simple upper/lower field vs runtime output across all 40,503 non-range, non-surrogate codepoints ≥ U+0020; SpecialCasing full-mapping override precedence. Not examined: none. **Ran:** a parser over `/opt/elixir/lib/elixir/unicode/UnicodeData.txt` comparing every simple `upper`/`lower` field to `String.upcase/downcase(_, :default)` → **1 downcase divergence (U+0130 → `69CC87` = i+U+0307) and 27 upcase divergences (U+1F80–U+1FF3, Greek precomposed-with-iota-subscript → decomposed full uppercase, e.g. U+1F80 → `1F08 0399`)**. Each divergence is a documented **unconditional** SpecialCasing full mapping — confirmed by reading `SpecialCasing.txt` above the `# Conditional Mappings` marker (`0130; 0069 0307; …`, `1F80; …; 1F08 0399;`, `1FF3; …; 03A9 0399;`). Zero unexplained transcription errors.

- **UCD decomposition/combining-class parsing (lines 157–176)** — the `decompositions` and `combining_classes` maps are built but not enforced by this file. Basis: `verified` dead-code. Axes examined: tree-wide consumer search; normalization delegation path. Not examined: the byte-level transcription of the two maps — **moot**, because they have no runtime consumers (see caveat 2). **Ran:** `grep -rn "decompositions\|combining_classes"` over `/opt/elixir/lib/elixir/` returns only the binding site (line 109); `String.normalize/2` (`lib/string.ex:862–886`) and `String.equivalent?/2` (`:805`) delegate to `:unicode.characters_to_nfd/nfc/nfkd/nfkc_binary`; `security.ex:111,113` and `tokenizer.ex:474,545` call `:unicode.characters_to_nfd_list`/`nfc_list` — none consume this module's maps.

- **`compute_lookup` ASCII exclusion (`Map.delete(prefixes, "")`, line 241)** — single-byte mappings dropped from generated clauses, handled by the generic ±32 catch-alls (lines 335, 453). Basis: `verified`. Axes examined: ASCII upper/lower via generic clause vs non-ASCII via generated clause. Not examined: none — the full-table diff above exercises every ASCII and non-ASCII code point; ASCII `upcase("i")`→`I` and `downcase(<<0x41,…>>)`→lowercase both route correctly.

## Risks / caveats

- **[Resolved — verified] Case-mapping table transcription.** The first pass hedged that it "did not exhaustively diff every generated case-mapping entry." Executed this pass: all 40,503 non-range, non-surrogate code points ≥ U+0020 diffed; only the 28 expected SpecialCasing full-mapping divergences appear, each confirmed unconditional in `SpecialCasing.txt`. No open assumption remains.

- **[Resolved — verified benign] `decompositions` / `combining_classes` (line 109) and `String.Break.decompose/2` (line 564) are dead code.** Tree-wide grep shows the two bindings are referenced only at line 109, and `String.Break.decompose/2` has no source caller anywhere (all `decompose` hits are `Macro.decompose_call`/`decompose_args` in unrelated modules, or the compiled definition in `Elixir.String.Break.beam`). Normalization is OTP-delegated. Benign unused code, not a runtime defect; out of scope for a security review. No finding warranted.

## Did I verify this

Yes — by execution on the in-repo Erlang/OTP 29 + Elixir 1.21.0-dev toolchain, and by reading the callgraph neighbours named in the standing context.

- **Full case-mapping table diff (executed).** Parsed `unicode/UnicodeData.txt`, compared every simple `upper`/`lower` field to `String.upcase/downcase(_, :default)`: 1 downcase + 27 upcase divergences, all documented unconditional SpecialCasing full mappings (cross-checked against `SpecialCasing.txt`). Converts the "did not exhaustively diff" hedge to `verified`.
- **Runtime behaviour (executed).** Ran the downcase/upcase invalid-UTF-8, Turkic, Greek-sigma, trim_leading/trailing, split, and normalization/equivalence probes listed inline above; all outputs match the ruled-out reasoning with no crash.
- **Dead-binding / callgraph check (executed).** Grepped `/opt/elixir/lib/elixir/` for `decompositions`, `combining_classes`, and `decompose`; read `lib/string.ex` (`normalize`/`equivalent?` → OTP `:unicode.*`), `unicode/security.ex` (`characters_to_nfd_list`), and `unicode/tokenizer.ex` (`characters_to_nfc_list`) to confirm normalization does not route through this module's maps.

I did not modify any files. No axis is left on `Not examined` — the only transcription I deliberately did not byte-diff (the `decompositions`/`combining_classes` maps) is moot dead code with no runtime consumer, verified by grep, so no untried axis of it can affect behaviour.

## Dropped in re-review

None. The first pass reported no findings, so none were dropped. Both hedged caveats were preserved and are now `verified`/benign with executed evidence, per the "prefer downgrading" rule; neither was upgraded to a finding because no evidence of a defect was found.

## ⚠ Auto-restored ruled-out entries (retained on record)

The two entries below are preserved verbatim intent from the prior report; both are now folded into the executed ruled-out list above with `Not examined: none` backed by runs. They remain dismissed.

- **`String.Break.trim_trailing` multibyte look-behind (lines 504–554), including `size < @whitespace_max_size` short path** — see ruled-out entry above; `verified` by executed `trim_trailing`/`trim` runs.
- **UCD parsing obligations relevant to this file** — see the two UCD ruled-out entries above; case-mapping parsing `verified` by the full-table diff, decomposition/combining-class parsing `verified` as dead code.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 2 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **UCD case-mapping / decomposition parsing obligations relevant to this file** — canonical-vs-compatibility distinction (`<<h,_>> when h != ?<`, line 160) keeps only canonical mappings; `Canonical_Combining_Class` parsed numerically (line 173); `First/Last` and `..` ranges expanded (`expand_range`, `rangify`); SpecialCasing unconditional mappings keystore-merged (lines 201–221). Basis: `verified` — see the full-table diff under "Did I verify this". Axes examined: whole UnicodeData.txt simple upper/lower field vs runtime output (2000+ mappings), SpecialCasing full-mapping override precedence, range-row expansion via observable normalization. Not examined: the `decompositions`/`combining_classes` maps built at line 109 — but these are **dead** (see caveat 1), so their transcription is moot.

- **`downcase/upcase` crash or mis-handling on invalid/partial UTF‑8** — the generated multibyte clauses all fall through to the byte-at-a-time catch-alls (`downcase(<<byte, rest::bits>>, …)` at line 334, `upcase/3` at line 452) plus the `""` terminators (lines 342, 460), so any binary is total. Basis: `verified`. Axes examined: empty binary, lone continuation byte, truncated lead byte, invalid two-byte sequence, ASCII-around-garbage. Not examined: none — `String.downcase/upcase` on `<<>>`, `<<0x80>>`, `<<0xC2>>`, `<<0xFF,0xFE>>`, `<<0x41,0x80,0x5A>>` all returned without error, correctly touching only ASCII (`<<65,128,90>>` → `<<97,128,122>>`).
