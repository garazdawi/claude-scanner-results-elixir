# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/integer.ex
<!-- sha256: eb82d80bd85fd570f4cddd589f336a8bbb1198f176e52f96187057a027a029aa -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:13:03Z_

Verified the algorithmic behavior with measurements. Writing the report.

## 1. Summary

`Integer` is a leaf numeric-utility module with almost no security surface (no code/atom/command/file/network/deserialization sinks); the one meaningful issue is a quadratic-time blowup in the newly-added `Integer.popcount/1` on large dense integers, exploitable as a CPU-exhaustion DoS when the integer is attacker-influenced.

## 2. Security

#### `CSCAN-integer-popcount-quadratic-dos`

- **Finding ID:** `CSCAN-integer-popcount-quadratic-dos`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `integer.ex:54-58`, public `popcount/1` and private `popcount/2` (the loop `defp popcount(n, acc), do: popcount(n &&& n - 1, acc + 1)` at line 58)
- **Class:** Resource consumption (algorithmic complexity, CWE-407 / CWE-400)
- **Issue:** `popcount/1` counts set bits with Kernighan's algorithm (`n &&& (n - 1)`), performing one iteration per *set bit*, and each iteration executes two full-width bignum operations (`n - 1` and `&&&`) over the entire integer. For a large dense integer this is O(bits²). The input is a plain `non_neg_integer`, so a large dense integer is a fully in-contract argument, not a spec violation.
- **Impact / threat:** An application that computes a Hamming weight on an integer derived from untrusted input (a bitset/bitmask from a client, a big integer decoded from JSON/a binary protocol, or a value from `Integer.parse`/`String.to_integer`) can be driven to consume a full CPU core for seconds-to-minutes with a single modestly-sized value. Measured: a ~800,000-bit all-ones integer takes ~15.7 s in one call; extrapolating quadratically to the BEAM's integer ceiling (~3.3M bits) gives ~minutes per call. Because the cost tracks the population count, not the magnitude, the input can be compact (~100–240 KB) — a strong amplification. A handful of concurrent calls saturates every scheduler, producing an availability DoS. The trust boundary crossed is untrusted runtime data → a stdlib function that a caller reasonably expects to be cheap. (The function is new in 1.20.0, so few callers yet guard it.)
- **Evidence:** `verified`.
  - Quadratic scaling — `elixir -e '... Integer.popcount((1 <<< k) - 1) ...'`:
    ```
    popcount of 100000-bit all-ones: 331.6 ms
    popcount of 200000-bit all-ones: 751.2 ms
    popcount of 400000-bit all-ones: 3474.9 ms
    popcount of 800000-bit all-ones: 12661.0 ms      (≈4× per doubling)
    ```
  - Cost is the algorithm, not the magnitude — same-size sparse vs dense integer:
    ```
    sparse (1 set bit,  ~800k-bit int): 0.03 ms
    dense  (800k set bits, same size ): 15758.95 ms
    slowdown factor: 562820x
    ```
  - A linear alternative produces the identical result on the same input in ~125 ms (≈126× faster and O(n), so the gap widens with size):
    ```
    linear byte-table popcount of 800k set bits: 125.07 ms => 800000
    ```
- **Suggested fix:** Replace the per-set-bit bignum loop with a single linear pass over the integer's byte representation, e.g. convert once with `:binary.encode_unsigned/1` and sum a compile-time-generated per-byte popcount lookup (the same `for … defp …` clause-generation idiom already used for `count_digits_nosign/3` in this file). That is O(n) in the integer's size, keeps the small-integer case fast, and removes the quadratic blowup on dense inputs.

### Ruled out

- **`pow/2` unbounded exponent (amplification).** `Integer.pow(base, exponent)` (`integer.ex:148-151`) computes `base ** exponent`; a tiny exponent yields a huge bignum. *Verified* bounded by the BEAM's integer system limit: `Integer.pow(10, 1_000_000)` returns a ~405 KB result in ~46 ms, and `Integer.pow(10, 2_000_000)` raises `SystemLimitError` rather than allocating without bound. Per-call cost/memory is bounded (~tens of ms, sub-MB), so no meaningful amplification — not a finding.
- **`parse/2` cost / exception on very long digit strings.** The module's own `count_digits` scan is linear; the superlinear part is `:erlang.binary_to_integer/2` (a BEAM builtin), which is *verified* bounded by the same system limit: ~1M digits parses in ~88 ms, and ≥~2M digits raises `SystemLimitError` instead of grinding. Cost is proportional to input with no amplification and is capped. (Caveat: on such huge inputs `parse` raises `SystemLimitError` rather than returning `:error` — noted below.)
- **`undigits/2` bignum build (`integer.ex:304-305`).** Horner accumulation `acc * base + digit` is O(n²) in the digit count, but this is inherent to base conversion and is bounded by the same integer system limit (raises `SystemLimitError` past the ceiling). No amplification beyond input size; not a distinct finding.
- **Error-message interpolation** (`"invalid base #{inspect(base)}"` line 355; `"invalid digit #{digit} in base #{base}"` line 302). Interpolated values are integers (or `inspect/1`-escaped) landing in an exception message string — no downstream interpreter, no injection.
- **Unicode/confusable digits in `parse`.** *Verified* `count_digits_nosign` matches only ASCII `0-9`/`A-Z`/`a-z`; `Integer.parse("١٢٣")` (Arabic-Indic digits) returns `:error`. No Unicode-digit confusability — a positive property.
- **Missing guards on `mod/2`, `floor_div/2`, `ceil_div/2`, `to_string/2`, `to_charlist/2`.** Non-integer / out-of-range-base arguments raise (`ArithmeticError` / `badarg` / `FunctionClauseError`) as documented; reaching those requires violating the documented type contract (out of scope).
- **Compile-time clause generation** (`digits = [...]` + `for` at lines 380-390). Generates `count_digits_nosign/3` clauses from a module-local literal only — no filesystem/network/external input at build time.
- **Dead defensive code.** All guard clauses (`popcount` negative branch, `parse` `base not in 2..36`, `undigits` invalid-digit clause, `count_digits_nosign` fallback) are reachable from the public API with ordinary inputs; none are unreachable.

## 3. Risks / caveats

- **Reachability of the finding depends on the caller.** `Integer.popcount/1` becomes a DoS only when an application feeds it an attacker-influenced large integer. That is a realistic but specific usage (Hamming weight on externally-supplied bitsets/big integers); I could not survey callers from this single file. The defect — the O(bits²) algorithm — is intrinsic to this file regardless.
- **Scheduler impact not precisely characterized.** BEAM bignum arithmetic is reduction-counted and yields, so a single `popcount` call consumes a full core but should not hard-freeze the VM; I verified wall-clock cost, not scheduler-monopolization. The DoS is CPU-exhaustion under concurrency, not a single-call VM freeze.
- **Worst-case near the integer ceiling is extrapolated.** I measured up to 800K bits (~15.7 s) directly; the ~minutes-per-call figure at the ~3.3M-bit BEAM limit is a quadratic extrapolation, not a direct run (to avoid a multi-minute command).
- **Minor robustness note (not scored):** `Integer.parse/2` is documented to return `{integer, remainder} | :error`, but on pathologically large digit strings (≥~1.5–2M digits) it raises `SystemLimitError` instead. A caller treating `parse` as total could be crashed by a ~2 MB+ all-digit input; impact is a single bounded process crash with no amplification, so I did not raise it as a security finding.
- Tested against the in-tree build (`Elixir 1.21.0-dev`, `Erlang/OTP 29`) compiled from the source under review; behavior on other OTP versions (different bignum limits) may shift the exact thresholds but not the quadratic shape.

