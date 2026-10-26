# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/set.ex
<!-- sha256: 4bac170c9919334f2ca2b4db77a4c072731e7f5b2e70887f40ad44b666846396 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/set.ex
<!-- sha256: 4bac170c9919334f2ca2b4db77a4c072731e7f5b2e70887f40ad44b666846396 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T13:00:00Z (hedge-clearing pass)_

## Summary

`Set` is a thin, deprecated polymorphic-dispatch layer whose only security-relevant behavior is that every public function dispatches directly to a module name taken from the input map's `__struct__` field, with no allow-list or behaviour gate. This pass resolves the two first-pass hedges: the caller-reachability caveat is now backed by an executed tree-wide caller survey (zero in-tree callers), and the "no weaponizable gadget" assertion is upgraded to an explicit low-confidence finding because it cannot be exhaustively verified.

## Security

#### `CSCAN-set-struct-module-dispatch`
- **Finding ID:** `CSCAN-set-struct-module-dispatch`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `set.ex:20-27` (`target/1` macro), reached from every public function (`delete/2` `set.ex:31`, `to_list/1` `set.ex:133`, `size/1` `set.ex:116`, `member?/2` `set.ex:106`, `put/2`, `union/2`, `intersection/2`, `difference/2`, `disjoint?/2`, `subset?/2`, `equal?/2`, `empty/1`).
- **Class:** Protocol and behaviour dispatch (dispatch on externally-influenced type/module name).
- **Issue:** The `target/1` macro extracts `module` from the `%module{}` pattern (i.e. the map's `__struct__` key) and every public function then calls `module.<fun>(set, …)` directly. There is no allow-list, no `function_exported?/3` check, and no behaviour/protocol gate — unlike a real Elixir protocol, which only dispatches to modules that provided a `defimpl`. Whatever atom sits in `__struct__` is invoked as a module, with the attacker-crafted map passed as the first argument. A caller who forwards a struct-shaped map derived from untrusted data turns `Set.*` into a "call `Mod.fun(my_map[, caller_value])` for `fun ∈ {delete, difference, disjoint?, empty, equal?, intersection, member?, put, size, subset?, to_list, union}` on any loaded module `Mod` of the attacker's choice" primitive.
- **Impact / threat:** The trust boundary crossed is *deserialization / external map construction*: any path that produces a map with an atom `__struct__` from outside (a `:erlang.binary_to_term` result, an external term store, an atom-keyed decoder) and then reaches a `Set.*` call. Because the module only needs to be already loaded (not newly created), the gadget survives even the "safe" deserialization path (`binary_to_term(bin, [:safe])`), which a developer may believe neutralizes type confusion. Practical impact is bounded — the attacker controls the module and the map (arg 1) but not the second argument, and no single obviously-weaponizable stdlib sink matches these names/arities (see `CSCAN-set-dispatch-gadget-absence` below) — so this is a type-confusion / weak call-gadget and unexpected-code-path concern rather than a direct RCE, and it lives in a deprecated module. Hence Low.
- **Evidence:** `verified`.
  - `elixir /tmp/set_test.exs` → `Set.to_list(%{__struct__: Evil, secret: "data"})` printed `!! Evil.to_list/1 was invoked as a dispatch target with arg: %{__struct__: Evil, secret: "data"}` and returned `[:pwned]`; `Set.size(%{__struct__: Evil})` invoked `Evil.size/1`; `Set.to_list(%{__struct__: Map, a: 1, b: 2})` dispatched to `Map.to_list/1` → `[a: 1, b: 2, __struct__: Map]`.
  - `elixir /tmp/set_test2.exs` → a `%{__struct__: Map, a: 1}` round-tripped through `:erlang.binary_to_term(bin, [:safe])` still dispatched: `safe-deserialized dispatch: [a: 1, __struct__: Map]`. Non-struct maps raise a controlled `ArgumentError - unsupported set: %{a: 1}`, and a non-atom `__struct__` raises a controlled `ArgumentError` (no crash escalation).
- **Suggested fix:** The module is deprecated; the real fix is to stop using it and call `MapSet` directly. If it must remain, gate the dispatch to a known-set allow-list (e.g. only dispatch when `module in [MapSet, HashSet]` or a behaviour check), so an arbitrary `__struct__` cannot select the dispatch target.

#### `CSCAN-set-dispatch-gadget-absence`
- **Finding ID:** `CSCAN-set-dispatch-gadget-absence`
- **Severity:** Low
- **Confidence:** low
- **Location:** `set.ex:20-27`, all dispatch sites (as above).
- **Class:** Protocol and behaviour dispatch (gadget reachability).
- **Status:** **Upgraded from hedged assumption in first pass.** The first pass asserted "no obviously-weaponizable stdlib sink matches these names/arities" and flagged it as "an asserted absence over the stdlib, not an exhaustive proof." That assertion cannot be verified — the gadget space is every module *loaded in the victim VM at call time* (application deps, OTP, and any dynamically-loaded module), which is unbounded and application-specific and therefore cannot be exhaustively enumerated from this file or from any fixed neighbour set.
- **Issue:** Severity of `CSCAN-set-struct-module-dispatch` is capped at Low *only if* no loaded module exports a `delete/2`, `put/2`, `to_list/1`, `size/1`, `member?/2`, `union/2`, `intersection/2`, `difference/2`, `subset?/2`, `disjoint?/2`, `equal?/2`, or `empty/1` that performs a dangerous side effect when handed an attacker-shaped map as arg 1. If such a function exists in a commonly-loaded dependency, the same primitive escalates toward arbitrary side-effect / RCE. I did not find one among the modules I inspected, but I hold this as a low-confidence non-finding rather than a verified absence.
- **Impact / threat:** If a matching gadget exists, escalates `CSCAN-set-struct-module-dispatch` from type-confusion to a stronger call-gadget; otherwise inert.
- **Evidence:** `asserted` (absence not provable within scope).
- **Suggested fix:** Same as `CSCAN-set-struct-module-dispatch` — allow-list the dispatch target; that closes the gadget-escalation path regardless of what is loaded.

### Ruled out
- **Caller reachability from untrusted input (in-tree survey):** Executed a tree-wide caller survey with `grep -rEn "\bSet\.[a-z]" lib | grep -vE "MapSet|HashSet"` → **0 matches**. There are no in-tree callers of the deprecated `Set.*` API anywhere under `lib`, so no in-repo code path feeds it any input (trusted or not). This *narrows* but does not eliminate the first-pass caveat: an external application that adds `Set.*` calls over deserialized data remains the only reachability vector, and that code is not in this checkout and cannot be enumerated. The finding therefore stays Low (ceiling of what the file enables), now on evidence rather than assumption. Note: the harness `Grep` tool under-reported here (returned 0 for `MapSet` while `grep` found 10+ files); I cross-checked with `grep` and relied on it.
- **Atom creation (CWE-770):** No `String.to_atom`, `binary_to_atom`, `Module.concat`, or similar in the file. `%module{}` reads an *existing* atom out of the input; it creates none. (Verified by reading — no such call sites exist in `set.ex`.)
- **`inspect(set)` in `unsupported_set/1` (`set.ex:162`):** Only reached on the `set ->` fallback, i.e. values with **no** `__struct__` key (plain maps / non-maps), so it cannot be a struct with a hostile custom `Inspect` impl; `inspect` also applies its default `limit: 50`, so no unbounded-output DoS. Verified: `Set.to_list(%{a: 1})` produced `unsupported set: %{a: 1}` with no side effects.
- **Crash / DoS on malformed input:** Non-struct input and non-atom `__struct__` both terminate in a controlled `ArgumentError` (verified above); no unbounded recursion or quadratic loop — the fallbacks are single `Enumerable.reduce` passes over the caller's own sets.
- **Mixed-type reduce fallbacks (`difference/2`, `disjoint?/2`, `intersection/2`, `union/2`, `subset?/2`, `equal?/2`):** These use `Enumerable.reduce`, which is gated by the `Enumerable` protocol's `defimpl`; a missing impl raises `Protocol.UndefinedError`. No additional externally-influenced sink beyond the dispatch already reported. Note the fallbacks still call `target1.delete/2`, `target1.put/2`, `target1.member?/2`, `target2.member?/2`, and `target1.new/0` on the attacker-chosen module — these are already covered by `CSCAN-set-struct-module-dispatch`/`CSCAN-set-dispatch-gadget-absence` and add no new class.
- **Command/file/path/network/crypto/EEx/Base/URI/Regex sinks:** None present in this file.

### Did I verify this?

- **Dispatch primitive (`CSCAN-set-struct-module-dispatch`):** Yes — executed (`/tmp/set_test.exs`, `/tmp/set_test2.exs`), results quoted above. `verified`.
- **Caller reachability (first-pass hedge "I could not survey callers / single-file review"):** Now verified for this checkout — executed `grep` caller survey returned zero standalone `Set.*` call sites across `lib`. The residual unknown (external application callers over deserialized data) is out-of-checkout and genuinely unenumerable; it is reflected as the severity ceiling, not as a cleared fact. Hedge removed; replaced with executed evidence plus an explicit statement of the residual scope.
- **Gadget absence (first-pass hedge "I did not exhaustively hunt for a fully-weaponized gadget"):** Not verifiable — the gadget space is the set of modules loaded in an arbitrary victim VM, which cannot be exhaustively checked from this file. Hedge removed and upgraded to the explicit low-confidence finding `CSCAN-set-dispatch-gadget-absence` rather than left as prose.
- **Dispatch targets:** Read `map_set.ex` (the intended `%MapSet{}` target) to confirm the legitimate polymorphic contract; `hash_set.ex` present as the second historical target. Neither changes the dispatch analysis — both are ordinary struct modules and the macro does not restrict to them.
- **Standing-context "callgraph neighbours":** The neighbour list in the folder threat model covers the Erlang compiler front-end (`elixir_tokenizer`, `elixir_interpolation`, etc.) and is unrelated to `set.ex` (a `lib/*.ex` polymorphic-dispatch shim). Reading those modules cannot verify anything about this file's callers or dispatch gadgets, so I did not treat them as relevant neighbours; the relevant neighbours are the dispatch targets (`map_set.ex`/`hash_set.ex`) and the callers (surveyed above).

### Dropped in re-review
- None. Both first-pass findings/caveats are preserved: `CSCAN-set-struct-module-dispatch` unchanged; the two prose hedges are converted (reachability → evidence-backed Ruled-out entry; gadget absence → low-confidence finding `CSCAN-set-dispatch-gadget-absence`).
