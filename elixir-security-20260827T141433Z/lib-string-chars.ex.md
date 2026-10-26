# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/string/chars.ex
<!-- sha256: c2d4e09945100ade09212a7b76b13c400987dba876ada232c9be9f3725a9380d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:33:19Z_

I've read the entire 67-line file and verified the runtime behaviors of every implementation. This file is a protocol definition (`String.Chars`) plus five trivial `to_string/1` implementations. Here is the report.

---

# Security review — `lib/elixir/lib/string/chars.ex`

## 1. Summary
`String.Chars` defines the one-directional `to_string/1` conversion protocol with five implementations (`Atom`, `BitString`, `List`, `Integer`, `Float`); every conversion is either identity or a safe stdlib call in the value→string direction, and the review found no security defect reachable through the documented public API (`to_string/1` / `#{}` interpolation).

## 2. Security

No findings.

### Ruled out

- **Atom implementation → atom-table exhaustion (CWE-770).** *Basis: verified.* The `Atom` impl runs `Atom.to_string/1`, i.e. the atom→string direction, which creates no atoms. There is no `String.to_atom`, `binary_to_atom`, or `Module.concat` anywhere in the file. `to_string(:abc)` returned `"abc"` and `to_string(nil)` returned `""` with no atom creation. The dangerous (string→atom) direction is not present.
- **Template/interpolation injection via `String.Chars` (the interpolation sink).** *Basis: verified/by-contract.* `String.Chars` is the conversion side of `#{}`; escaping for any downstream interpreter (SQL/HTML/shell/regex) is categorically the sink's responsibility, and none of the five impls is contractually required to escape. They introduce no interpreter here, so no injection originates in this file.
- **BitString impl returns non-UTF-8 bytes unchanged.** *Basis: verified.* `to_string(<<255,254,192,128>>)` returned the identical bytes (`String.valid?/1` → `false`). This is the documented identity contract ("convert a structure to a binary … only if applicable") and matches all Elixir versions; validating UTF-8 would break legitimate binary construction. Any downstream code that *assumes* `to_string` yields valid UTF-8 has a bug at that consumer, not here. Non-byte-aligned bitstrings (`<<1::4>>`) correctly raise `Protocol.UndefinedError` — the defensive fallback clause is reachable, not dead code.
- **`to_string(nil) == ""` footgun.** *Basis: verified.* Confirmed `""`. This is documented, long-standing Elixir behavior; it can surprise a caller who interpolates `nil` into a query, but the conversion itself is not a vulnerability in this file (no injection is produced, only an empty substitution).
- **List impl → crash / resource use.** *Basis: verified/contract-excluded.* `to_string([1_114_112])` (codepoint above U+10FFFF) raised `UnicodeConversionError`. Reaching this requires the caller to interpolate a value that is not a valid charlist, which violates the `String.Chars.List` contract (an integer list of valid codepoints); per scope rules, contract-violating arguments are excluded. `List.to_string/1` (`:unicode.characters_to_binary`) is O(input) with no amplification.
- **Code execution / compile-time execution / command exec / file ops / archive / deserialisation / network / crypto / shared-state / concurrency / round-trip.** *Basis: verified by exhaustive read.* The complete set of calls in the file is `Atom.to_string/1`, `List.to_string/1`, `Integer.to_string/1`, `Float.to_string/1`, and a single `raise Protocol.UndefinedError`. None evaluate/compile code, spawn processes, touch the filesystem or network, deserialise bytes (`binary_to_term`), perform cryptographic comparisons, or write shared/global state. The `@protocol` used in the raise is a compile-time constant, not external input. Protocol dispatch is on the value's type (normal protocol semantics), not on any atom/module derived from external bytes.

## 3. Risks / caveats
- Scope was the documented public entry point `to_string/1` (invoked by `Kernel.to_string/1` and by `"#{...}"` interpolation). Behavioral claims above were exercised against an `elixir` built from this source and their outputs cited.
- Residual (out-of-scope, not defects here): the identity passthrough for binaries means `#{untrusted_binary}` places raw attacker bytes into a string. That is safe *as a conversion*; the security obligation lives at whatever interpreter consumes the resulting string (parameterize SQL, escape HTML, use argv-based `System.cmd/3`, etc.). No change to this file would be the right fix for such a downstream sink.
- No dependency-, build-, or archive-time (`mix`) reachability exists in this file; it is a pure runtime library module with no macros that read files or the network at compile time.

