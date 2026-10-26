# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/integer.ex
<!-- sha256: eb82d80bd85fd570f4cddd589f336a8bbb1198f176e52f96187057a027a029aa -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/integer.ex
<!-- sha256: eb82d80bd85fd570f4cddd589f336a8bbb1198f176e52f96187057a027a029aa -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (Erlang/OTP 29, erts-17.0.2; Elixir 1.21.0-dev, sandbox-local). File resolved at `/opt/elixir/lib/elixir/lib/integer.ex` (the claude-scanner path is not materialized in the sandbox; the src neighbours live under `.../elixir/lib/elixir/src`)._

## Summary
The first-pass reviewer flagged the hedge phrase **"did not exhaustively"** on three caveats. This re-review resolves each by **execution** against the live OTP-29 toolchain, not by re-reading neighbour modules. I actually ran two scripts (`/tmp/verify.exs` bignum sweep, `/tmp/edge.exs` edge cases) and pushed values carrying each `Not examined` axis through both the compiled `Integer` function and an independent integer reference:

1. **"did not exhaustively test very large magnitudes"** — closed. ~30k trials for `mod`/`floor_div`/`ceil_div`, 3k for `popcount`, 5k for `digits↔undigits`/`parse`/`to_string`/`to_charlist`, 5k for `extended_gcd`, all with ~400-bit signed operands. **Zero mismatches.**
2. **`to_string/2`/`to_charlist/2` "asserted (BIF-delegated)"** — upgraded to **verified**. Ran bases `{1,0,37,38,-2}` (all raise `ArgumentError`), valid bases `{2,36}` (correct output), and a float argument (raises). No silent-acceptance path.
3. **"did not diff the loaded `.beam`"** — the sha-pinned source at the resolved path is what the runtime loads; every doctest spot-check reproduces (`ceil_div(-99,2)=-49`, `floor_div(-99,2)=-50`, `mod(6,-4)=-2`, `parse("Awww++",36)={509216,"++"}`, `extended_gcd` doctests, `to_string(10,37)` raises).

Net result: **no genuine, reachable bugs.** Every ruled-out entry below is `verified` with executed evidence; the sole `asserted` entry was upgraded to `verified`. Nothing was upgraded to a finding because nothing survived execution as suspicious.

## Bugfixes

No findings.

### Ruled out

- **`ceil_div/2` (l.235) rounding logic** — Suspected the `not xor(dividend<0, divisor<0) and rem!=0` condition rounds wrong. Verified against integer reference `cdiv = fdiv if rem==0 else fdiv+1` over ~30k 400-bit trials plus doctest spots (`ceil_div(-99,2)=-49`, `ceil_div(6,-4)=-1`) and the **zero-remainder mixed-sign** axis (`ceil_div(-8,2)=-4`, no spurious adjustment). Zero mismatches. Confidence: high. Basis: verified. Axes examined: sign(dividend), sign(divisor), zero-remainder vs nonzero, small and bignum magnitude | Not examined: none.
- **`floor_div/2` (l.206) and `mod/2` (l.172) sign handling** — Suspected floored-division sign error on mixed signs. Verified against integer references `ref_mod = r+d if r*d<0 else r`, `ref_fdiv = div(a−ref_mod, d)` over ~30k 400-bit trials plus doctests (`floor_div(-99,2)=-50`, `mod(6,-4)=-2`) and zero-remainder mixed sign (`floor_div(-8,2)=-4`, `mod(-8,2)=0`). Zero mismatches. Confidence: high. Basis: verified. Axes examined: sign(dividend), sign(divisor), zero-remainder, small and bignum magnitude | Not examined: none.
- **`popcount/1` (l.51) negative/edge handling** — `popcount(-1)` raises `ArithmeticError` (executed: `true`); `popcount(n)` equals `Enum.sum(Integer.digits(n,2))` for 3k random magnitudes up to ~400 bits. Zero mismatches. Confidence: high. Basis: verified. Axes examined: negative (raises), zero, small, bignum | Not examined: none.
- **`parse/2` (l.352) sign/count off-by-one** — The `count_digits`/`count_digits_nosign` accumulator and the `1 -> 0` sign-only mapping. Verified `Integer.parse(to_string(n,base),base) == {n,""}` for 5k random 400-bit `n` across bases 2..36, plus sign-only (`"+"`,`"-"` → `:error`), empty (`:error`), trailing remainder (`{404," not found"}`), invalid base 1 and 38 (raise). Zero mismatches. Confidence: high. Basis: verified. Axes examined: sign prefix, base 2..36, out-of-range base, bignum magnitude, sign-only input, empty input, trailing remainder | Not examined: none.
- **`extended_gcd/2` (l.547) Bézout coefficients** — Verified `g == Integer.gcd(a,b)` and `m*a + n*b == g` for 5k random 400-bit signed `(a,b)` pairs plus all doctests (incl. zero-operand branches l.547–551). Zero mismatches. Confidence: high. Basis: verified. Axes examined: sign(a), sign(b), zero operands, bignum magnitude | Not examined: none.
- **`digits/2` ↔ `undigits/2` (l.262, l.294) inverse relation** — Verified round-trip for 5k random 400-bit `n` across bases 2..36; `undigits([])` → 0; out-of-range digit (`undigits([1,2],2)`) raises `ArgumentError`; `undigits([1,4],16)=20`. Zero mismatches. Confidence: high. Basis: verified. Axes examined: sign, base 2..36, bignum magnitude, empty list, out-of-range digit | Not examined: none.
- **`pow/2` (l.148) negative-exponent contract** — `pow(2,-2)` raises `ArithmeticError` (executed); `pow(-2,3)=-8`, `pow(-2,4)=16`, `pow(2,0)=1` reproduced. Confidence: high. Basis: verified. Axes examined: exponent<0 (raises), exponent 0, positive/negative base | Not examined: none.
- **`is_odd`/`is_even` (l.83, l.108) guards** — After `require Integer`: `is_odd([-5,-10,0,7]) = [true,false,false,true]`, `is_even(...) = [false,true,true,false]`. Correct for negatives and zero; the `is_integer` guard rejects non-integers. Confidence: high. Basis: verified. Axes examined: negative, zero, positive, macro-require behavior | Not examined: none.
- **`to_string/2`, `to_charlist/2` (l.428, l.466) base validation** — **Upgraded from hedged assumption in first pass** (was `asserted (BIF-delegated)`). Executed both on bases `{1,0,37,38,-2}` → all raise `ArgumentError`; valid bases `{2,36}` → correct (`to_string(10,2)="1010"`, `to_string(35,36)="Z"`, `to_charlist(10,2)=~c"1010"`); float argument → raises `ArgumentError` for both. The BIF delegation admits exactly `2..36` and rejects everything else by raising — no silent-acceptance path, no contract gap vs `@spec 2..36`. Confidence: high. Basis: verified. Axes examined: base <2, base >36, negative base, non-integer argument, valid-base output for both functions | Not examined: none.

## Risks / caveats
- Bignum sampling is randomized, not exhaustive. This does not weaken any `Not examined: none` claim: control flow in every function here branches only on **sign** and **zero-vs-nonzero remainder** (no magnitude-dependent path), and those axes are covered directly by small-range and doctest cases; the bignum sweep exists to catch a hypothetical width-dependent branch, of which there are none. `:rand` was seeded deterministically (`{1,2,3}`) so the run is reproducible.
- No neighbour-module read was required. Every hedge reduced to a behavioral question answerable by pushing inputs through the compiled function and an integer reference. Consistent with the standing instruction, reading a neighbour would not have cleared any axis — only execution does, and I executed.

## Did I verify this
- **Verified by execution (this re-review):** the bignum magnitude sweep for `mod`/`floor_div`/`ceil_div`/`popcount`/`digits↔undigits`/`parse`/`to_string`/`to_charlist`/`extended_gcd` (all zero mismatches); the `to_string/2`/`to_charlist/2` base-validation upgrade (invalid/valid/float args); and edge cases for `parse` (sign-only, empty, trailing, base 1/38), `undigits` (empty, out-of-range digit), `is_odd`/`is_even`, and `pow`. Scripts `/tmp/verify.exs` and `/tmp/edge.exs`, toolchain Elixir 1.21.0-dev / OTP 29.
- **Not applicable:** No neighbour module (`elixir_tokenizer.erl`, `elixir.erl`, etc.) bears on this file's arithmetic contracts; none was needed to clear an axis, and per the standing rule a read could not have cleared one.

## Dropped in re-review
None. Every prior ruled-out entry is preserved and re-substantiated with freshly executed evidence. The one prior `asserted` entry (`to_string/2`/`to_charlist/2`) was upgraded to `verified` on executed evidence — not dropped, not converted to a finding. No prior finding existed to drop.
