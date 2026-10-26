# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/list/chars.ex
<!-- sha256: 41c0911fe6a5c9c69b88c1e97ae189b99315b2631542ece5ccd8c93777d09f39 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:35:13Z_

## Summary

`List.Chars` is a small, stable protocol file whose five implementations (`Atom`, `BitString`, `List`, `Integer`, `Float`) all behave as documented, and I found no genuine reachable bugs.

## Bugfixes

No findings

### Ruled out

- **`Atom.to_charlist(nil)` returns `~c""` instead of `~c"nil"`** — Not a bug. This is an intentional, long-standing special case mirroring `String.Chars`/`to_string(nil) == ""`. It is deliberate, not an oversight, and no doc/spec here promises otherwise. Basis: `verified` — ran `List.Chars.to_charlist(nil)` → `[]` (empty charlist); the `to_charlist(nil)` head at chars.ex:25 precedes the general atom head at :27, so it is reached and returns the documented-consistent empty charlist.
- **`BitString` impl raises on non-binary bitstrings (`<<1::4>>`)** — Not a bug. `@spec to_charlist(t) :: charlist` has no sensible charlist result for a sub-byte bitstring; raising `Protocol.UndefinedError` with a clear description ("cannot convert a bitstring to a charlist") is the correct contract, and the message names the offending value. Basis: `verified` — ran `List.Chars.to_charlist(<<1::4>>)`, observed `protocol List.Chars not implemented for BitString, cannot convert a bitstring to a charlist / Got value: <<1::size(4)>>`.
- **`List` impl returns the list unchanged without validating it is a proper charlist** — Not a reportable bug. Passing a non-charlist list (e.g. `[:a, :b]`) violates the caller's contract (`t` implementing the protocol with a `charlist` result); the language does not enforce `@spec` at runtime and this pass-through inlining is intentional (see the comment at chars.ex:47 about the rewrite rule). Out of scope per the "caller violates documented contract" exclusion. Basis: `verified` — `List.Chars.to_charlist([1,2])` → `[1, 2]`.
- **`Integer`/`Float` impls** — delegate directly to `Integer.to_charlist/1` and `Float.to_charlist/1`; outputs `~c"3"` and `~c"1.0"` match the `charlist` spec. Basis: `verified` — ran both, observed `~c"3"` and `~c"1.0"`.

## Risks / caveats

The file is 62 lines; I reviewed every clause and exercised all five implementations plus the `BitString` raise path against a source-built `elixir`. The only behaviors that could surprise a caller (`nil → ~c""`, sub-byte bitstring raising, non-charlist lists passing through untouched) are intentional and consistent with `String.Chars`, so none rise to a bug. No doctests exist in this file to contradict the code, and the `@spec` matches every implementation's return shape.

