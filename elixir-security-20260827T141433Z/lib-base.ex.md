# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/base.ex
<!-- sha256: 8b5d6f6b12bbb908d23a0d943de38e3badec4f7acec0decf8813d6138c4522cd -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:03:33Z_

Linear scaling confirmed (no quadratic blowup). I have everything I need — the investigation is complete.

---

# Security review — `base.ex` (`Base` module)

## 1. Summary
The `Base` encoders/decoders and the new `valid*?` predicates are robust against crashes and behave consistently, but the base64/base32 decoders accept **non-canonical encodings** (multiple distinct inputs decode to the same bytes) even with default strict options — a round-trip-integrity hazard for callers who trust the encoded form.

## 2. Security

#### `CSCAN-base-noncanonical-decode`
- **Finding ID:** `CSCAN-base-noncanonical-decode`
- **Severity:** Low
- **Confidence:** high
- **Location:** `base.ex:1138-1223` (`decode64base!`/`decode64url!` final-quantum clauses, e.g. `bsr(unquote(decode_name)(c2), 4)::2` at line 1139); `base.ex:1784-1866` (`decode32*!` tails, e.g. line 1785); validators that bless the same forms at `base.ex:1034-1104` and `base.ex:1682-1750`.
- **Class:** Round-trip integrity
- **Issue:** For the final encoded character of a quantum, only the significant high bits are used; the remaining low bits are **discarded with `bsr`** rather than being required to be zero. So the decode is not injective: many different encoded strings map to the same output binary, even under the default `padding: true` and no `ignore`. The `valid64?`/`valid32?`/`hex_valid32?`/`url_valid64?` predicates only check that each byte is in-alphabet and that the padding *shape* is right — they never check the pad bits, so they return `true` for the non-canonical variants too. base16 is unaffected (every bit is significant).
- **Impact / threat:** Untrusted bytes reach these functions through the documented public API. If a caller builds a security decision on the *encoded* representation — a MAC/signature verified over the base64 text, a dedup/idempotency/cache key, an allow-list of encoded tokens, or a "nonce already seen" set — an attacker can submit a non-canonical variant that passes the encoded-form check yet decodes to the same (or a neighbouring attacker-chosen) value, bypassing the control. Conversely, a value stored decoded and later re-encoded will not match the attacker's original string. Exploitability depends on downstream usage, hence Low; but the property is invisible to a developer who reasonably assumes base64/base32 decoding is canonical. RFC 4648 §3.5 (Canonical Encoding) permits but does not require discarding these bits, and recommends rejecting non-zero pad bits where distinct-encoding integrity matters — so this is a security hardening gap, not an RFC violation.
- **Evidence:** `verified` — compiled the file under review to `/tmp/basetest` and ran with `-pa` (`:code.which(Base)` → `/tmp/basetest/Elixir.Base.beam`):
  - `Base.decode64("QQ==") → {:ok, "A"}`, `Base.decode64("QR==") → {:ok, "A"}`, `Base.decode64("Qf==") → {:ok, "A"}` (13 distinct 4-char strings decode to the byte `"A"`; 52 accepted `Q?==` variants collapse onto 4 output bytes).
  - `Base.encode64(Base.decode64!("QR==")) → "QQ=="` (re-encoding is non-canonical), and `Base.valid64?("QR==") → true`.
  - `Base.decode32("AB======") → {:ok, <<0>>}` (same byte as `"AA======"`), `Base.valid32?("AB======") → true`.
- **Suggested fix:** For strict decoding, verify the discarded bits are zero instead of masking them — in the base64 2-char tail require `band(decode(c2), 0x0F) == 0`, in the 3-char tail `band(decode(c3), 0x03) == 0`, and the analogous `0x03`/`0x0F` masks for the base32 tails — raising `ArgumentError` otherwise, and mirror the checks in the `validate*?` predicates. If lenient behaviour must be kept for compatibility, document the non-canonical acceptance and/or gate strict rejection behind an option.

### Ruled out
- **`valid*?` disagreeing with its decoder (validation oracle).** `verified` — 2,000,000+ differential trials across all bases with `[]`, `padding: false`, `case: :mixed`/`:lower` produced **0** cases where `valid?/2` and `{:ok,_} <- decode/2` disagreed. The predicates faithfully predict decode success (they share the alphabet/pad-shape logic; the SWAR 7-byte ranges match the per-char guards exactly).
- **Decoder crash / non-`:error` exception leak (DoS).** `verified` — 1,000,000 fully-random hostile-byte inputs (incl. bytes >127, whitespace, all option combos) through every `decode*/2`: **0** exceptions escaped the `rescue ArgumentError`; each returned `{:ok,_}` or `:error`. The segment-split math (`div(byte_size+7,8)-1`) never produces a negative/oversized size, and odd-length base16 returns `:error`.
- **Quadratic / super-linear blowup.** `verified` — decode/validate/whitespace-ignore time scales linearly from 250 KB→2 MB (decode 419→3746 µs, valid 110→813 µs, ws-decode 1834→14835 µs). Encoders/decoders use the binary-append accumulator optimisation; validators are tail-recursive.
- **Atom-table exhaustion (CWE-770).** `asserted` from full-file read — every `String.to_unsafe_atom` site (lines 439, 620–623, 738, 1004–1009, 1317, 1656–1665) runs at **compile time** over a fixed `[upper: …, lower: …]` list; no runtime path converts caller input to atoms (`:case`/`:padding`/`:ignore` are compared, never created).
- **Invalid `:ignore`/`:case` option value raising `FunctionClauseError`/`CaseClauseError` instead of `:error`.** `verified` behaviour (`decode64("Zg==", ignore: :nope)` leaks `FunctionClauseError`; `decode16("41", case: :nope)` leaks `CaseClauseError`), but **excluded by scope**: it requires passing an option value outside the documented `@spec` (`ignore: :whitespace`, `case:` one of the listed atoms), i.e. a caller contract violation, and the impact is only that the caller crashes its own call.
- **`padding: false` / `ignore: :whitespace` malleability.** These are documented, opt-in options; the library offering them is by design. The residual malleability they add is folded into the finding above.

## 3. Risks / caveats
- I reviewed and exercised the exact source under review by compiling `base.ex` to `/tmp/basetest` and loading it with `-pa` (confirmed via `:code.which(Base)`); results are not from the pre-installed build.
- Fuzzing is probabilistic: 2M differential and 1M crash trials with 24–40 byte inputs give high confidence for short/medium strings but do not prove the absence of a mismatch on some rare longer structure; the SWAR range arithmetic was additionally checked by hand (no cross-lane carry; ranges equal the per-char alphabets).
- The malleability finding's real-world severity is a function of **downstream** usage not visible in this file; a project that treats `Base`-encoded strings as canonical identifiers or verifies signatures over the encoded form should treat it as higher than Low.
- I did not fetch the RFC 4648 text in-session; the §3.5 characterisation reflects its well-established "MAY/should reject non-zero pad bits" reading, and the finding stands on the verified behaviour regardless of RFC framing.

