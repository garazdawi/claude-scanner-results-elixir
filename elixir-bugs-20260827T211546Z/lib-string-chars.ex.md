# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/string/chars.ex
<!-- sha256: c2d4e09945100ade09212a7b76b13c400987dba876ada232c9be9f3725a9380d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:45:29Z_

## Summary
`String.Chars` is a small, canonical standard-library protocol with implementations for `Atom`, `BitString`, `List`, `Integer`, and `Float`; all documented public paths behave as specified and I found no reachable bugs.

## Bugfixes

No findings

### Ruled out

- **`to_string(nil)` returns `""` rather than `"nil"`** — This is the deliberate `Atom` special-case clause (`chars.ex:30-32`) and matches Elixir's long-standing documented behavior; `Atom.to_string(nil)` is only reached for non-`nil` atoms. Not a defect. Basis: `verified` — `to_string(nil)` printed `""` and `to_string(:foo)` printed `"foo"`.
- **`BitString` impl raises instead of returning a string** — The second clause (`chars.ex:44-49`) raises `Protocol.UndefinedError` for non-binary bitstrings. The `@spec` describes the success shape (`String.t()`); raising on a value with no valid string representation is the intended contract, and the raise carries a clear message identifying the offending value. Basis: `verified` — `to_string(<<1::4>>)` raised with message `"...cannot convert a bitstring to a string..."` naming `<<1::size(4)>>`.
- **`List` impl on improper/non-codepoint lists** — `to_string(charlist)` delegates to `List.to_string/1` (`chars.ex:53`), which raises on invalid charlists; this is documented `List.Chars`/`List.to_string` contract behavior, not a defect in this file, and requires input already violating the charlist contract. Basis: `asserted` for the failure path; `verified` that a valid charlist `[?a,?b]` returns `"ab"`.
- **`Integer`/`Float` impls** — Direct delegation to `Integer.to_string/1` and `Float.to_string/1` (`chars.ex:57-65`), both total over their input types and matching the `String.t()` spec. Basis: `verified` — `to_string(123)` → `"123"`, `to_string(1.5)` → `"1.5"`.
- **`@spec to_string(t) :: String.t()` vs implementations** — Every success-returning clause yields a binary; raising clauses do not violate a success spec. No Dialyzer contract mismatch. Basis: `asserted` (spec inspection) corroborated by the `verified` runtime returns above.

## Risks / caveats
The review covered every clause of all five `defimpl` blocks and the protocol definition — the file's complete public surface. No doctests are present, so there was no executable-documentation contract to falsify beyond the moduledoc/`@doc` prose, which matches observed behavior. The `latin1` VM-encoding warning is environmental (locale), not a defect in this file.

