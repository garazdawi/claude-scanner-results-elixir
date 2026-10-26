# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/json.ex
<!-- sha256: a95392c6f3503cb4d80a3eac99554eb3f1a738421b7f0d5f9e6a076637494f18 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/json.ex
<!-- sha256: a95392c6f3503cb4d80a3eac99554eb3f1a738421b7f0d5f9e6a076637494f18 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28 (Erlang/OTP 29, erts-17.0.2; Elixir 1.21.0-dev). Every claim below is backed by code executed in this sandbox, not by an assumed invariant._

## Summary

`JSON` is a thin wrapper over Erlang's `:json` module plus a set of `JSON.Encoder` protocol implementations. Re-running the full public surface (`decode/1`, `decode/3`, `decode!/1`, `encode!/2`, `encode_to_iodata!/2`, `protocol_encode/2`, `__deriving__/2` and every `defimpl`) against live inputs, I found no genuine reachable bug. The first-pass hedges have been resolved by execution where the axis could be pushed through both implementations, and left as an honest coverage boundary where it could not.

## Bugfixes

No findings.

## Risks / caveats

- **Coverage.** I reviewed the entire file: the `JSON.Encoder` protocol (`__deriving__/2`, `fields_to_encode/2`, every `defimpl`) and the `JSON` module. The parsing/number/string *correctness* of tokens lives inside Erlang stdlib `:json` — that is a different component, not a callgraph neighbour in this folder's standing context, so I did not re-derive its internals. What I *did* verify by execution is that this wrapper marshals `:json`'s results and error tuples faithfully (see the two resolved hedges below).

- **RESOLVED HEDGE — decode/1 uses `:json`'s built-in number defaults, not the handlers documented for `decode/3`; are they equivalent?** `decode/1` calls `decode(binary, :ok, [])`, so after `Keyword.put_new(.., :null, nil)` only `%{null: nil}` is passed and `:json` applies its own float/integer/string defaults. The docs for `decode/3` (lines 398–400) name `&String.to_float/1` / `&String.to_integer/1` / `&Function.identity/1` as those defaults. I pushed the divergent axis — exponent/large/negative-zero/subnormal/big-int numbers — through **both** paths and compared outputs:
  - `decode/1` (Erlang defaults) and `decode/3` with the documented handlers returned identical values for `1e10, 1E10, 1.5, 123, 1.0e10, -0, 0.0, 1.5e-10, 10000000000000000000, 9223372036854775808`, and both returned `{:error,{:unexpected_sequence,0,"1.0e999"}}` for `1e999`.
  - The key normalization claim is now executed, not assumed: `:json` rewrites `"1e10"`→`10000000000.0` (adds the decimal point) *before* invoking the float handler, so the documented `&String.to_float/1` default never raises on a valid JSON exponent number. Confirmed. No mismatch on any tried numeric axis. Confidence: high (executed).

- All doctests (`decode/1`, `decode!/1`, `encode!/1`, `encode_to_iodata!/1`) reproduce documented output exactly (re-run this pass).

### Ruled out

- **`decode/3`'s three-clause `catch` is non-exhaustive → an `:error` class escapes and breaks the `{:error,_}` contract.**\
  Fuzzed ~30 malformed inputs this pass (`01`, `-`, `1e`, `+1`, `[1,]`, `{"a":}`, `"\x"`, `"\ud800"`, lone/paired surrogates, `Infinity`, `NaN`, `[1,2] x`, `[1,2,\xFF]`, `<<0xFF>>`, `""`, `"   "`, `[`, `{`, `tru`, `nul`, `{"a":1,}`, …); every case returned a proper `{:error, reason}` in `{:unexpected_end | :invalid_byte | :unexpected_sequence}` — no `rescue`/`catch` fired. I additionally pushed the user-supplied-handler axis: a handler that raises (`float: fn _ -> raise "boom" end`) is swallowed by `:json` and surfaces as `{:error,{:unexpected_sequence,0,"1.5"}}`, while a *wrong* documented-shaped handler (`integer: &String.to_float/1` on `"123"`) raises `ArgumentError` that escapes the `catch` — but that is the caller supplying a broken decoder, which is the documented contract of `decode/3`, not a wrapper defect; the `decode/1` path cannot reach it. Basis: verified for tried axes. **Axes examined:** the 3 documented `:json` error classes, all handler slots via `decode/1`'s empty-list path, deep-nested errors, custom-handler raise/return. **Not examined:** proof that no *fourth* undocumented `:json` `:error` class exists for some exotic valid-looking input — absence cannot be established by fuzzing. Because that axis is untried, this entry stays **asserted**, not verified. Confidence: medium.

- **`decode/1` reports a valid trailing-whitespace byte as `:invalid_byte` (naming the space, not the following non-JSON char).**\
  Re-ran both implementations: `JSON.decode("[1,2] x")` → `{:invalid_byte, 5, 32}`; bare `:json.decode("[1,2] x")` → `{:invalid_byte, 32}` — same byte value, so this wrapper mirrors Erlang's canonical decoder. The offset `byte_size(binary)-byte_size(rest)` is precisely the first byte after the complete top-level value (`"true false"`→`{:invalid_byte,4,32}`, `"123 456"`→`{:invalid_byte,3,32}`). Consistent, documented-shape result. Ruled out as not-a-defect. **Axes examined:** whitespace-trailing, non-ws-trailing, multi-token trailing, comparison vs bare `:json`. **Not examined:** none. Basis: verified.

- **`__deriving__` with `:only` / `:except` / bare / empty-list.**\
  Built structs for each variant. `only:` emits exactly the listed fields; `except:` drops the excluded; bare `@derive` emits all; and re-confirmed this pass that `only: []` yields `"{}"` (the `io == []` guard at line 74). Field/var hygiene in the generated `def encode(%{...}, encoder)` resolves correctly. **Axes examined:** only, except, bare, empty-only. **Not examined:** none. Basis: verified.

- **`offset/1` returns wrong/zero position.**\
  Errors at varying depth resolve via `error_info.cause.position`: top-level (`<<0xFF>>`→0), mid-array (`[1,2,\xFF]`→5), and deep-nested (`[1,2,{"k":"\ud800"}]`→20, executed this pass). `unexpected_end` reports `byte_size(binary)`; the `else -> 0` fallback only triggers when no `error_info` is present. **Axes examined:** depth 0 / mid / deep, all three error classes. **Not examined:** none. Basis: verified.

- **`JSON.Encoder.Map.key/2` non-string keys / `protocol_encode` clause ordering.**\
  Re-executed: integer (`1`→`"1"`), float (`1.5`→`"1.5"`), atom (`:atom`→`"atom"`) and binary keys are stringified per the documented `%{String.Chars.t() => _}` contract; a tuple key (no `String.Chars` impl) raises `Protocol.UndefinedError` — the documented contract, not a bug. `protocol_encode`'s `%{} = value when not is_map_key(value, :__struct__)` correctly routes plain maps to `JSON.Encoder.Map` before the struct fallthrough. **Axes examined:** int/float/atom/binary/no-impl keys, map-vs-struct dispatch order. **Not examined:** none. Basis: verified.

### Did I verify this?

- Toolchain present and used: Erlang/OTP 29, Elixir 1.21.0-dev. Every "verified"/"executed" claim above was produced by running Elixir against the actual module, not by reading `:json`.
- The one hedge I could not fully discharge by execution — whether a *fourth* undocumented `:json` error class could escape `decode/3`'s `catch` — is left explicitly on `Not examined`, and that ruled-out entry is therefore recorded as **asserted** (medium confidence), per the rule that reading a neighbour cannot clear an untried axis and absence cannot be fuzzed to a proof.
- The `:json`-internal token correctness remains an out-of-component boundary (Erlang stdlib, not a neighbour module in this folder's standing context); I verified the wrapper's marshaling of its outputs and errors, which is the part this file owns.

### Dropped in re-review

None. All prior ruled-out entries and caveats are preserved; two first-pass hedges (numeric default-handler equivalence, and the `:json`-mirroring of trailing-byte reporting) were upgraded from asserted-by-reasoning to executed-and-verified, and the `catch`-exhaustiveness entry was downgraded from `verified` to `asserted` to reflect the one axis that cannot be tried.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`__deriving__` with `:only` / `:except` / no-option / empty list.** Ruled out. Built structs for each variant; `only` emits exactly the listed fields, `except` drops the excluded ones, bare `@derive` emits all fields, and `only: []` correctly yields `"{}"`. Field/var hygiene in the generated `def encode(%{...}, encoder)` resolves correctly. Basis: verified.
