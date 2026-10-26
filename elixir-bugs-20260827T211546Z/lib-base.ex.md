# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/base.ex
<!-- sha256: 8b5d6f6b12bbb908d23a0d943de38e3badec4f7acec0decf8813d6138c4522cd -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:41:14Z_

Based on extensive differential testing, I have a complete picture. Here is my report.

# Bug-hunting review — `base.ex`

## Summary
After exhaustive differential fuzzing and running the full doctest suite against a build of the file, I found no genuine, reachable bugs in `Base`'s documented public API — the SWAR-optimized validators, the encoders, and the decoders are mutually consistent, match a strict independent reference, and match the separately-built upstream `Base`.

## Bugfixes

No findings

## Risks / caveats
- **Coverage of what I tested.** I compiled the file under review (`Code.require_file`) and exercised every public entry point (`encode16/decode16/valid16?`, `encode64/decode64/valid64?`, `url_*64`, `encode32/decode32/valid32?`, `hex_*32`) across `:upper`/`:lower`/`:mixed` and `padding: true|false`:
  - **Doctests:** all 65 pass (`ExUnit ... doctest Base`).
  - **Roundtrip + validator agreement** (20k random inputs × 8 variants): `decode(encode(x)) == {:ok, x}` and `valid?(encode(x)) == true` — no failures.
  - **Corruption fuzz** (200k × 10 variants, one byte mutated to any 0–255 value, so both the ASCII gate and per-char rejection in the SWAR path are exercised): `decode` success ⇔ `valid?` — no disagreement.
  - **Random alphabet+`=` strings** (500k × 7 variants, lengths that exercise the 7-byte SWAR stride and every padding-quantum clause): `decode` success ⇔ `valid?` — no disagreement.
  - **Strict independent reference** for base64 (padding required) and base32 (padding required), 1M samples each: zero discrepancies against a hand-written RFC-4648 validator.
- **The one apparent discrepancy I chased was not a bug.** Under `padding: false`, `valid64?`/`decode64` *accept* a string that still carries trailing `=` (e.g. `"bC=="` → `{:ok, "l"}`). This is intentional "ignore padding" behavior and is **identical** in the independently-built upstream `Base` (`/opt/elixir/.../Elixir.Base.beam`), and `valid?` stays consistent with `decode`, so it is not a defect.
- **Verified vs. hand-reasoned.** The SWAR constants (`@swar_ge_*`/`@swar_gt_*`) were additionally checked by hand against each alphabet's byte ranges (`ge_lo = 0x80-lo`, `gt_hi = 0x7F-hi`, no cross-lane carry given the `band(w,MASK80)==0` gate), and the encode-clause bit-widths/pad counts against RFC 4648 quantum sizes; both are consistent with the differential results.
- **Out of scope / not treated as bugs (spec violations).** Passing a `:case`/`ignore` option outside the documented set (e.g. `Base.decode64("...", ignore: :foo)` → `FunctionClauseError` from `remove_ignored/2`; `Base.encode16("x", case: :bad)` → `CaseClauseError`) or a non-binary first argument raises rather than returning `:error`. These require violating the functions' documented `@spec`/`@doc` contracts, so per the review scope they are not reported. Note only `decode64/2` and `url_decode64/2` guard `is_binary` on the public head, while `decode16/2`, `decode32/2`, `hex_decode32/2` reach the guard one level deeper — a cosmetic asymmetry with no behavioral difference for documented (binary) inputs.
- I did not attempt to independently re-derive the moduledoc alphabet tables cell-by-cell; their correctness is corroborated indirectly by the roundtrip and doctest results, which would fail on any wrong alphabet entry.

