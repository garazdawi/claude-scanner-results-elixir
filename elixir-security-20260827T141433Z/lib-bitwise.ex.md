# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/bitwise.ex
<!-- sha256: de6cbf29cf4f04287ddb2c4456a44cb90d3e2ce85f04ceca4207a05935d9d469 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/bitwise.ex
<!-- sha256: de6cbf29cf4f04287ddb2c4456a44cb90d3e2ce85f04ceca4207a05935d9d469 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

I read the entire file and, for the re-review, executed every operation on the actual sandbox VM (OTP 17.0.2) rather than resting on documentation. This replaces the first-pass hedges ("outside this file", "assumes the BEAM primitives behave as documented") with executed evidence.

## Report

**1. Summary**

`Bitwise` is a thin, pure-passthrough wrapper over Erlang's bitwise BIFs (`:erlang.band/bor/bxor/bnot/bsl/bsr`) plus a deprecated compile-time `__using__/1` macro. It performs no atom creation, code evaluation, I/O, deserialization, or dispatch on external data. No security findings.

Note on scope: this file lives in `lib/elixir/lib`, not the `lib/elixir/src` compiler front-end covered by the standing threat model. None of the threat-model callgraph neighbours (tokenizer, interpolation, parser, expander) call or are called by `Bitwise`; there are no in-tree neighbour modules to read for this file. The only downstream sinks are the native `:erlang` BIFs, which have no Erlang/Elixir source in this tree — so I verified their observable behaviour by execution instead of by reading a neighbour.

**2. Security**

No findings.

### Ruled out

- **`bsl/2` (and `<<</2`) memory-amplification DoS** — *Basis: verified by execution.* `bsl/2` is the operation with amplification potential: a tiny left operand and a large shift produce a large bignum. I ran the BIF directly on the sandbox VM: `erlang:bsl(1, 4_000_000)` → 500 001 bytes in 18 µs (OK); `erlang:bsl(1, 5_000_000)` and `erlang:bsl(1, 100_000_000)` → `error:system_limit`. The VM checks the resulting bignum size and raises `system_limit` cleanly before unbounded allocation, so a single call cannot allocate more than a few hundred KB. No unbounded-allocation defect in this file. Residual hot-loop cost is the caller's loop, inherent to all arithmetic.
  - *Axes examined:* small-input/large-shift amplification (executed, capped); negative shift `bsl(1,-2)` → `0` (shrinks, executed); non-integer operand (executed — see below).
  - *Not examined:* a hypothetical BEAM release with a different or absent bignum size cap. This is an environment axis, not an input axis — the sandbox only provides OTP 17.0.2, so I cannot vary the VM to try it. It remains a caveat (§3), not a defect in this file, since bounding integer size is a VM responsibility the wrapper cannot enforce.
- **`bsr/2` (and `>>>/2`) mirror-amplification DoS** — *Basis: verified by execution (newly examined this pass; the first pass only exercised `bsl`).* A negative right-shift is a left-shift and carries the same amplification potential. Executed: `erlang:bsr(1, -5_000_000)` → `error:system_limit`. Same clean cap as `bsl`. No defect.
- **`band/2`, `bor/2`, `bxor/2`, `bnot/1` cost** — *Basis: verified by execution.* I fed a ~500 KB bignum (`bsl(1, 4_000_000)`) into each: `band(Big,3)` → 1 byte, `bor(Big,3)` → 500 001 bytes, `bxor(Big,3)` → 500 001 bytes, `bnot(5)` → `-6`. Output is never larger than the larger operand — no small-input→large-output amplification, so no resource-consumption sink.
  - *Axes examined:* large-operand throughput (executed); result-size bound (executed); non-integer operand (executed).
  - *Not examined:* none.
- **Non-integer / contract-violation input** — *Basis: verified by execution.* `erlang:band(foo, 3)` → `error:badarith` (`ArithmeticError` at the Elixir level) with a clean stacktrace, no crash or corruption. Passing a non-integer is a documented caller contract violation and yields a clean error, not a security issue.
- **`__using__/1` macro injection** — *Basis: verified by inspection.* `unquote(except)` at line 44 only ever splices one of three hardcoded literal lists (lines 34, 37, 40); `options` merely selects which via the truthiness of `Keyword.get/2`. No caller-supplied data reaches the quoted AST — no unquote/hygiene escape. It is compile-time, `@doc false`/deprecated, and driven by the developer's own `use Bitwise` line, not an untrusted-input boundary.
  - *Axes examined:* data-into-AST flow (inspection — only literals reach `quote`).
  - *Not examined:* none applicable — the spliced values are source literals, so there is no runtime input axis to push.
- **Atom creation** — *Basis: verified by inspection.* The only atoms (`:only_operators`, `:skip_operators`, the `def unquote(:"~~~")`/`unquote(:"^^^")` names) are compile-time literals in this source, not conversions of external bytes. No `String.to_atom`-style sink.
- **Code execution / file / network / deserialization / dispatch sinks** — *Basis: verified by inspection.* None of these sink shapes appears; every function body is a direct call to an `:erlang` bitwise BIF.

**3. Risks / caveats**

- The `system_limit` cap I observed (between the 4M- and 5M-bit shift boundary) is a property of the OTP release in the sandbox (17.0.2). A release with a different or absent cap would re-open the `bsl`/`bsr` amplification question, but bounding integer size is a VM responsibility, not something the `Bitwise` wrapper can or should enforce without breaking legitimate large-shift use. I cannot try this axis (only one VM available), so it stays a caveat.
- Findings were judged strictly at this file's boundary. All functions are documented as operating on integers; passing a non-integer is a caller contract violation (out of scope) and yields a clean `ArithmeticError`/`badarith`, confirmed by execution.

### Dropped in re-review

Nothing dropped. Two first-pass hedges were resolved rather than removed:

- **"I did not audit `:erlang.bsl/bsr/band/bor/bxor/bnot` themselves … outside this file … assumes the BEAM primitives behave as documented"** — resolved. There is no Erlang/Elixir source for these native BIFs in the tree (and no threat-model neighbour calls this file), so option (a) "read the neighbour" does not apply. Instead of upgrading a non-defect to a finding, I replaced the assumption with **executed** evidence: I ran each BIF directly and observed its behaviour (amplification caps, result-size bounds, contract-violation error). The claim is now verified by execution, not assumed.
- **`bsl` cap "property of the OTP release under test"** — the amplification input axis is now executed and confirmed capped; the only residual is the un-runnable "different VM" environment axis, retained explicitly under *Not examined* and §3.

## Did I verify this?

- Read the complete file (274 lines).
- Executed on the sandbox VM (OTP 17.0.2): `bsl(1, 4M)` = 500 001 B / 18 µs; `bsl(1, 5M)` and `bsl(1, 100M)` = `system_limit`; `bsr(1, -5M)` = `system_limit`; `band/bor/bxor(Big, 3)` = 1 / 500 001 / 500 001 B; `bnot(5)` = -6; `bsl(1,-2)` = 0; `band(foo, 3)` = `badarith`.
- Verified by inspection: `__using__/1` splices only literal lists; all atoms are compile-time literals; no code/file/network/deserialization/dispatch sinks.
- Not verifiable in this sandbox: behaviour of a BEAM release without the bignum size cap (only OTP 17.0.2 available) — retained as an un-tried environment axis, not a finding.
