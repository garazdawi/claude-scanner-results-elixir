# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/collectable.ex
<!-- sha256: 8231d3b5fbceefd1d35ef53d1e96018d513a75fcc88527912aad9b26226088f1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:54:47Z_

## Summary
The `Collectable` protocol and its `List`, `BitString`, and `Map` implementations are correct and total over their documented inputs; I found no genuine bugs.

## Bugfixes

No findings

### Ruled out

- **`BitString.into/1` binary clause — accumulator type juggling between iolist and bitstring.** The initial accumulator is `[binary]` (a list); it stays a list while every incoming element is a binary (`[acc | x]` builds an iolist, and `is_list/1` is true even for the improper lists produced), and switches to a bitstring the first time a non-binary bitstring arrives (third `:cont` guard converts the iolist via `IO.iodata_to_binary/1`, then concatenates). Every `:cont`/`:done` combination is covered by a guard, and byte order is preserved. Verified: `Enum.into([<<1::1>>, "a"], "x")` → `<<120, 176, 1::size(1)>>` and `Enum.into(["a","b"], "x")` → `"xab"`. Basis: `verified`.
- **`BitString.into/1` non-binary bitstring clause.** Any `is_bitstring(x)` element (binary or partial) concatenates correctly; a non-bitstring element raises `ArgumentError` as documented. Verified: `Enum.into([<<1::1>>, <<0::1>>], <<7::3>>)` → `<<30::size(5)>>`. Basis: `verified`.
- **`List.into/1` `:done` ordering and non-empty deprecation.** `list ++ :lists.reverse(list_acc)` yields the original list followed by the collected elements in input order; the non-empty case emits the documented `IO.warn` deprecation and still produces the historical result. Verified: `Enum.into([1,2,3], [0])` → `[0,1,2,3]` (with warning) and `Enum.into([1,2,3], [])` → `[1,2,3]`. Basis: `verified`.
- **`Map.into/1` shape handling.** `{:cont, {k, v}}` inserts via `Map.put/2`; any other `:cont` payload raises the documented `ArgumentError`. Covered by the clause structure; `Enum.into([a: 1, b: 2], %{})` → `%{a: 1, b: 2}`. Basis: `verified`.
- **Module `@doc`/doctests (lines 12, 37–42, 69).** The `MapSet` walk-through and `Enum.into` examples all evaluate to the shown results against the built compiler. Verified via `Enum.into` reproductions. Basis: `verified`.
- **`@spec into(t)` vs implementations.** Each `into/1` returns `{acc, fun/2}` where the function accepts `{:cont, term} | :done | :halt`; matches the `command` type and return spec. Basis: `asserted` (spec is structurally satisfied by all three impls, confirmed by observing the returned 2-tuples during the runs above).
- **`__acc` variable name (line 161).** Double-underscore prefix is a valid (unused) variable binding, not a typo affecting behavior. Basis: `verified` (module compiled and ran without warning on that clause).

## Risks / caveats
Review covered only `collectable.ex`; the actual insertion loop lives in `Enum.into_protocol/2` (enum.ex), which drives these collector functions and is out of scope. The `:halt` accumulator caveat documented in the moduledoc is a contract on implementers, not something this file can enforce. The latin1 VM-encoding warning in the sandbox is environmental and unrelated to the code under review.

