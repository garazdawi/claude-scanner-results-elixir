# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/collectable.ex
<!-- sha256: 8231d3b5fbceefd1d35ef53d1e96018d513a75fcc88527912aad9b26226088f1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:31:04Z_

All three concerns check out as safe: the error message is bounded (~1 KB via `inspect`'s default `limit: 50`, shown truncated with `...`), it's on a terminal raise path, and atom creation is zero. I have enough to write the report.

---

## Summary

`collectable.ex` defines the `Collectable` protocol and its `List`/`BitString`/`Map` implementations; it performs only shape-checked data insertion with safe (`inspect`-based) error messages and contains no security-relevant sinks reachable through its documented API.

## Security

`No findings`

### Ruled out

- **Quadratic blowup in the `BitString` collector (`into/1`, lines 143–190).** The collector appends via `<<acc::bitstring, x::bitstring>>` (line 150/174) and, in the binary case, transitions the accumulator from an iodata list to a bitstring once a non-byte-aligned element arrives (line 152–153) — a shape that classically defeats the ERTS append optimization. I suspected O(n²) reachable via `Enum.into(untrusted, <<>>)`. **Basis: verified.** Doubling the element count (`elixir /tmp/bs_bench3.exs`) gave 980→2437→6474→17614 µs across n=100k→800k (≈2.5× per doubling, ~n^1.4), whereas a genuinely O(n²) reference (`acc ++ [i]`) gave 118 ms→612 ms→2510 ms across n=10k→40k (~4–5× per doubling, and 3 orders of magnitude slower at 10× fewer elements). The bitstring path is sub-quadratic and finishes 800k elements in 17 ms; not a DoS vector, and the required input (millions of individual bitstrings collected into a bitstring) has no realistic untrusted-data path.
- **Atom-table exhaustion via `Map.into` keys (line 195, `Map.put(map_acc, key, value)`).** Keys arrive from the collected enumerable. **Basis: verified.** `Map.put` stores keys as-is with no `to_atom` conversion; driving the collector with a fresh non-atom string key showed `atom_count delta=0` (`/tmp/err_path.exs`). No atomization anywhere in the file.
- **Expensive/unbounded `inspect` in the raise clauses (lines 166, 185, 206).** The error paths interpolate `#{inspect(other)}` on an attacker-supplied element. **Basis: verified.** Feeding a 1,000,000-element list to the `Map` collector's error clause (`/tmp/err_path2.exs`) produced a 954-byte message (truncated at `inspect`'s default `limit: 50` with `...`), and the branch immediately raises and aborts the collection. Output is bounded and the cost is one-shot on a terminating path — no amplification.
- **Code execution / eval / apply / dynamic dispatch.** None present. The file contains no `Code.*`, `apply/3`, `Module.concat`, `String.to_atom`, `binary_to_term`, file/network/command sinks, or EEx/regex/template construction (full read of lines 1–212). Protocol dispatch is standard `Enum.into`/`for`-comprehension resolution on the *collectable's* type, chosen by application code; all three implementations here are safe regardless of which is selected.
- **`List` collector `:done` cost (line 133, `list ++ :lists.reverse(list_acc)`).** Called once at `:done`; O(len(list)+len(acc)), no per-element `++`. Not quadratic.
- **`IO.warn` deprecation notice (lines 119–125).** Static string with no interpolation of collected/external data; not a log-injection or format-string vector.

## Risks / caveats

- **Reproductions ran on the sandbox's Elixir 1.21.0-dev / OTP 29 (JIT) build.** The bitstring-append cost is a runtime (ERTS) property; on an older OTP without the current bitstring/append handling the same code *could* trend more steeply. The source-level code here is unchanged either way — the ruling reflects the behavior of the toolchain built from the source under review, which is what the review is scoped to.
- **Scope was this file only.** `Collectable` implementations for other types (e.g. `MapSet`, `IO.Stream`, `File.Stream`) live in their own modules and were not reviewed; a resource/TOCTOU concern in a *streaming* collectable (the `:halt`/`:done` resource-cleanup contract described in the moduledoc, lines 72–87) would surface there, not here.
- I did not find any input that violates only the documented public API (`Collectable.into/1`, and its use by `Enum.into/2` / `for ... into:`); findings requiring callers to pass values that break the `t`/`command` type contract were treated as out of scope per the ground rules.

