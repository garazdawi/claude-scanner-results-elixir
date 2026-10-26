# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/float.ex
<!-- sha256: b97cc0c148edae0bfae92f18d462c080ddf3695e694e129b80bb8e7c4ba1fc6b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:49:16Z_

I have everything I need. All facets are reproduced and root-caused. Writing the report.

---

## Summary

`Float.parse/1` is safe on its decimal/exponent paths, but its bare-integer path runs an unbounded `:erlang.binary_to_integer/1` on the whole input, which both burns superlinear CPU and raises an *uncaught* `SystemLimitError` — crashing callers that rely on the documented `{float, binary} | :error` contract.

## Security

#### `CSCAN-float-parse-unbounded-bigint`
- **Finding ID:** `CSCAN-float-parse-unbounded-bigint`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `lib/elixir/lib/float.ex:205-209` (`finish_mantissa/3`, the `_dot? = false` clause), reached from `parse/1` → `parse_unsigned/1` → `parse_mantissa/3` (`float.ex:189`).
- **Class:** Resource consumption (with an unhandled-exception / contract-violation facet).
- **Issue:** When the input is a run of digits with no decimal point and no exponent (optionally a leading `+`/`-`), parsing falls through to `finish_mantissa(binary, rest, false)`, which evaluates `:erlang.binary_to_integer(consumed(binary, rest)) * 1.0`. This builds the *exact* bignum for the entire consumed digit string before checking whether the value is even in float range. The clause rescues only `ArithmeticError` (the `* 1.0` overflow the comment on `float.ex:203-204` anticipates). It does **not** account for two behaviors of `binary_to_integer/1` on large input:
  - it raises `SystemLimitError` once the digit count exceeds an OTP limit (~1.3M digits) — a *different* exception class that escapes the `rescue`;
  - it spends superlinear time (~n^1.5–n^2) constructing the bignum for any large-but-sub-limit input, all of it wasted since the result immediately overflows to `:error`.

  The decimal and exponent paths do not have this problem: they delegate to `:erlang.binary_to_float/1`, which parses with bounded work and rejects out-of-range magnitudes early (verified: a 2 MB digit string with a `.` or an `e` returns `:error` in milliseconds).
- **Impact / threat:** `Float.parse/1` is the idiomatic way to coerce an untrusted string (form field, query param, CSV cell, config value) into a float, and its docs promise `:error` for anything un-coercible. A caller that forwards attacker-controlled text and matches `{f, ""}` / `_` — the documented pattern — cannot defend against this: a ~1.3 MB numeric string makes the call raise `SystemLimitError` and crash the calling process (per-request DoS in a Phoenix-style server; the 1.3 MB payload is within default body limits). Below that threshold, inputs of a few hundred KB to ~1.2 MB cost tens to ~130 ms of pure CPU per call, giving a request-flood amplifier. The OTP digit limit does cap the worst single call, so this is a bounded DoS, not unbounded amplification or memory corruption.
- **Evidence:** `verified`.
  - Uncaught crash, escaping the documented contract:
    ```
    $ elixir -e 'try do Float.parse(String.duplicate("9", 2_000_000)); rescue e -> IO.puts("RAISED: #{inspect(e.__struct__)}") end'
    RAISED: SystemLimitError -- a system limit has been reached
    ```
  - Idiomatic caller crashes instead of taking its `:error` branch:
    ```
    $ elixir -e 'case Float.parse(String.duplicate("9",1_500_000)) do {f,""} -> f; _ -> :bad end'
    ** (SystemLimitError) a system limit has been reached
        (elixir) lib/float.ex:206: Float.finish_mantissa/3
    ```
  - Root cause is `binary_to_integer` itself: `:erlang.binary_to_integer(String.duplicate("9",1_500_000))` → `SystemLimitError`.
  - Superlinear wasted CPU below the crash threshold (all return `:error`): `n=500000 → 28.5 ms`, `n=1000000 → 83.6 ms`, `n=1200000 → 130.5 ms`.
  - Contrast (safe paths): `Float.parse("9"×2_000_000 <> ".5")` → `:error`; `Float.parse("9"×2_000_000 <> "e5")` → `:error`; `Float.parse("1e" <> "9"×2_000_000)` → `:error`, all in milliseconds.
- **Suggested fix:** Restore the documented contract by broadening the rescue in the `_dot? = false` clause to also catch the system-limit error, e.g. `rescue [ArithmeticError, SystemLimitError] -> :error`. To also close the CPU-exhaustion vector, reject before conversion when the consumed integer carries more than ~309 significant digits (any such value overflows `max_finite` regardless) — stripping leading zeros first, since `Float.parse("0"×400 <> "34")` legitimately returns `{34.0, ""}`, so a naive byte-length guard would be wrong.

### Ruled out
- **`pow/2` overflow (`float.ex:120-121`)** — `:math.pow(2.0, 1_000_000)` raises `ArithmeticError`, but the guards force `base`/`exponent` to be numbers supplied by the caller, not untrusted strings; no external-data boundary is crossed. Verified it raises, but out of the untrusted-input scope.
- **`round/2`, `floor/2`, `ceil/2` precision handling (`float.ex:274-397`, `do_round` and helpers)** — precision is bounded to `0..15` by guards (`round p16`/`floor p16` verified to raise the documented `ArgumentError`), and the internal bignum arithmetic is bounded by the 52-bit mantissa × 10^15, so no external input controls cost or reaches an unguarded sink.
- **`ratio/1` on subnormals (`float.ex:589-627`)** — `Float.ratio(5.0e-324)` returns a ~300-digit denominator, but the input is a single 64-bit float, so the output size is fixed and bounded; no amplification.
- **`Float.parse/1` decimal and exponent paths (`float.ex:197-233`)** — both route through `:erlang.binary_to_float/1`, which rejects oversized magnitudes with `ArgumentError` (rescued) without building a bignum; verified `:error` in milliseconds on 2 MB inputs.
- **Deprecated `to_string/2`, `to_char_list/1,2` (`float.ex:695-705`)** — `@doc false` (not documented public API) and their `options` proplist is caller-supplied config, not attacker-forwarded data; out of the stated scope.
- **`to_string/1`, `to_charlist/1`, `max_finite/0`, `min_finite/0` (`float.ex:56-81, 657-689`)** — pure conversions of a single bounded float / compile-time constants; no external-input sink.

## Risks / caveats
- The `SystemLimitError` threshold (~1.3M digits) is an Erlang/OTP `binary_to_integer` limit observed on this OTP 29 build; the exact cutoff may differ on other OTP versions, but the *uncaught-exception* gap (only `ArithmeticError` is rescued) is version-independent and the CPU-cost facet exists on every version.
- Exploitability of the crash depends on the host application permitting a ~1.3 MB single string to reach `Float.parse/1`; the sub-threshold CPU cost is triggerable with smaller (few-hundred-KB) payloads and is the more broadly reachable facet.
- I reviewed only `float.ex`; whether `String.to_float/1` (referenced in the docs) or other callers share this pattern was not assessed, as it is out of file scope.

