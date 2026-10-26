# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/hash_set.ex
<!-- sha256: d6d1ab4cdf516045c62fb24bb2274c6004f15576e2ba6a1b1943b9615f55ad1a -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:13:27Z_

## Summary
The core tuple-trie set operations are correct, but every `Enumerable`/`Collectable`/`Inspect` protocol implementation reconstructs the module atom with `String.to_unsafe_atom("HashSet")` — which yields `:HashSet`, not the module `HashSet` (`:"Elixir.HashSet"`) — so all protocol-based use of a `HashSet` crashes with `UndefinedFunctionError`.

## Bugfixes

#### `CSCAN-hash_set-unsafe-atom-missing-elixir-prefix`
- **Finding ID** — `CSCAN-hash_set-unsafe-atom-missing-elixir-prefix`
- **Severity** — High
- **Confidence** — high
- **Location** — `lib/elixir/lib/hash_set.ex:282,288,294,309,328` (`Enumerable.HashSet.reduce/member?/count`, `Collectable.HashSet.into`, `Inspect.HashSet.inspect`)
- **Issue** — Each protocol impl does `module = String.to_unsafe_atom("HashSet")` to reference the deprecated module without triggering a compile-time deprecation warning, then calls `module.reduce/…`, `module.member?/…`, `module.size/…`, `module.put/…`, `module.to_list/…`. But the runtime atom of module `HashSet` is `:"Elixir.HashSet"`; `String.to_unsafe_atom("HashSet")` produces the unprefixed atom `:HashSet`, which is not a loaded module. Every such dispatch therefore fails. The correct incantation is `String.to_unsafe_atom("Elixir.HashSet")`, which is verified equal to `HashSet`.
- **Class** — Pattern match on the wrong shape / Contract violation (wrong module atom → undefined dispatch)
- **Impact** — Any documented protocol-based entry point into a `HashSet` crashes: every `Enum.*`/`Stream.*` call (via `Enumerable.reduce/count/member?`), `Enum.into/2` or `for … into: HashSet.new()` (via `Collectable.into`), and `inspect/1`/IEx display (via `Inspect.inspect`) raise `UndefinedFunctionError` for `:HashSet.*`. The direct API (`put`, `delete`, `member?`, `union`, `to_list`, etc.) works, but the entire protocol surface is dead. Because `inspect` is broken, even printing a `HashSet` at the shell fails (it degrades to an `Inspect.Error`).
- **Evidence** — `verified`.
- Probe: `String.to_unsafe_atom("HashSet")` → `:HashSet`; `HashSet == String.to_unsafe_atom("HashSet")` → `false`; `String.to_unsafe_atom("Elixir.HashSet")` → `HashSet`; `HashSet == String.to_unsafe_atom("Elixir.HashSet")` → `true`.
- Repro on `s = HashSet` with 3 elements:
  - `Enum.count/1` → `RAISED: function :HashSet.size/1 is undefined (module :HashSet is not available)`
  - `Enum.to_list/1` → `RAISED: function :HashSet.reduce/3 is undefined …`
  - `Enum.member?/2` → `RAISED: function :HashSet.member?/2 is undefined …`
  - `Enum.into/2` → `RAISED: function :HashSet.put/2 is undefined …`
  - `inspect/1` → returns an `#Inspect.Error<… function :HashSet.to_list/1 is undefined …>` instead of `#HashSet<[…]>`.
- **Suggested fix** — Change each `String.to_unsafe_atom("HashSet")` to `String.to_unsafe_atom("Elixir.HashSet")` (all five call sites), so the reconstructed atom is the actual module.

## Risks / caveats
- The direct functional API and all set algebra (`put`, `delete`, `member?`, `union`, `intersection`, `difference`, `subset?`, `disjoint?`, `equal?`, `size`, `to_list`) were fuzzed against `MapSet` — 2000 seeds × 200 mixed put/delete/member ops, plus 5000 seeds of union/intersection/difference/subset/disjoint/equal — with zero mismatches, including collision-heavy inputs that force sub-node creation and compaction. I found no correctness defect there.
- I could not fuzz the protocol impls against `MapSet` because they crash immediately; once the atom prefix is fixed, `reduce`/`into`/`inspect` should be re-exercised, but their logic is otherwise a thin delegation to the (correct) direct API.
- Whether this file was mutated from upstream is immaterial to the finding: the code as written breaks its own documented protocol implementations, and the fix is confined to the module-atom string.

### Ruled out
- **`union/2` size-guard branch selection** — the `size1 <= size2` head folds the smaller set into the larger; both orderings produce identical results. Verified: 5000-seed union fuzz vs `MapSet` matched exactly, including asymmetric sizes.
- **`delete/2` size underflow / `do_compact_node` promotion** — deleting absent elements returns the set unchanged (`:error` branch), and sub-node collapse/compaction preserves membership and size. Verified via the 2000-seed mixed-op fuzz (size and element-set tracked against `MapSet`, no mismatch).
- **`equal?/2` size-shortcut** — `size1 == size2` guard then `subset?`; matched `MapSet.equal?` across 5000 seeds. Verified.
- **`reduce/3` suspend/halt continuation plumbing** — exercised indirectly by `subset?`/`disjoint?` (which call `reduce` with `:halt`) across 5000 seeds with no mismatch; the direct `HashSet.reduce/3` is correct (the protocol wrapper's crash is the atom bug, not the reducer). Verified for the halt path; the `:suspend` path is only reachable via the (currently broken) `Enumerable` and was not independently exercised — asserted correct by inspection only, so noted as a residual gap rather than a firm ruling.

