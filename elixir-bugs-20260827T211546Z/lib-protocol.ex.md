# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/protocol.ex
<!-- sha256: 6bec90e7d0d905ced0c18f01dcf942945b53df9a0e50290555269a7bcd0807f4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:40:15Z_

I've completed a thorough two-phase review and verified every plausible candidate by running against the built Elixir 1.21.0-dev. All documented public-API paths behave per contract.

## Summary
A focused bug hunt of `protocol.ex` (protocol definition, consolidation, extraction, and impl/derive machinery) turned up no genuine, reachable defect in its documented public API; every plausible candidate was reproduced-against and found correct.

## Bugfixes

No findings

## Risks / caveats
- I could not exercise the deep type-checker signature path (`new_signatures/5`, using `Module.Types.Descr`/`Module.Types.Of`) end-to-end with a checker chunk present, only through `consolidate/2` on a real beam (which succeeded). A latent shape mismatch inside that path would surface only under the Dialyzer/type-checker consumer, not in dispatch, so it is outside what a black-box run can confirm. Nothing in the reading suggested a defect there.
- The deprecated default-argument path in `def/1` (lines 292–295) generates a bodyless fake head plus a fixed-arity real impl; it only emits a deprecation warning and is explicitly slated for removal, so any resulting arity quirk is by-design-deprecated behavior, not reported as a bug.
- `__derive__/3`'s `:lists.flatten(derives)` assumes `derives` is a list; a non-list `@derive` value would raise, but that is an internal (`@doc false`) entry point fed by the compiler, not a documented public call, so it is out of scope.

### Ruled out
- **`extract_matching_by_attribute/3` comprehension includes `nil`s** (lines 504–510): the `mod = extract_from_file(...)` binding acts as a truthiness filter in `for`, so `nil` results are dropped. Basis: `verified` — `for x <- [1,2,3], y = (if rem(x,2)==0, do: x), do: y` returned `[2]`, and `Protocol.extract_protocols([ebin])` returned no `nil` entries (`any nils in protocols: false`).
- **`:lists.seq(2, arity)` crashes for arity‑1 protocol functions** (line 282/289): `lists:seq(2,1)` returns `[]`, and a `def size(data)` protocol compiled and dispatched correctly. Basis: `verified` — `seq(2,1): []`; `MySize.size("abc") => 3`, `__protocol__(:functions) => [size: 1]`.
- **`impl_for`/fallback dispatch or `@fallback_to_any` wiring wrong**: fallback-to-`Any` returned `MySize.Any` for an int and `0` via the Any impl; non-fallback raising path is guarded correctly. Basis: `verified` — `impl_for int: MySize.Any`, `size int fallback: 0`.
- **`consolidate/2` contract deviation**: returned `{:ok, <binary>}` for a real protocol (`Enumerable`) and `{:error, :not_a_protocol}` for a non-protocol (`Enum`), matching the `@spec`. The `{:error, :no_beam_info}` seen for a script-defined protocol is the documented no-beam case, not a bug. Basis: `verified`.
- **`assert_protocol!/1` raising the wrong exception type**: returns `:ok` for a protocol and raises `ArgumentError` for a loaded non-protocol, per `@doc`. Basis: `verified` — `non-protocol raises: ArgumentError`.
- **`extract_protocols/1` and `extract_impls/2` doctests fail**: both pass against Elixir's own ebin. Basis: `verified` — `Enumerable in mods: true`, `List in imods: true`.
- **`__impl__/4` option parsing / `Keyword.merge`+`pop_lazy` mishandling `:for` or extra opts**, **`change_impl_for` clause ordering (struct vs `is_map`)**, and **`consolidate/5` Any-removal (`List.delete` vs `--`)**: read-verified against the dispatch order and the `types = if fallback_to_any?, ...` logic; struct clause precedes the `is_map` built-in clause so structs never mis-dispatch, and Any is included in `types` iff `@fallback_to_any`. Basis: `asserted` (logic), corroborated by the `verified` fallback/dispatch run above.

