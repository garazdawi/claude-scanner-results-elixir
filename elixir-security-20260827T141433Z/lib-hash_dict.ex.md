# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/hash_dict.ex
<!-- sha256: 773c7fc386a615c78e7f7a82a3dbc01378551aade240f3e255ae6949152d4dee -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:57:53Z_

The core tree operations are correct — 50 randomized put/delete/pop/fetch runs stayed perfectly consistent with `Map`, so there's no planted logic bug in the algorithms. The collision test confirms the deep-nesting mechanism: two full-`phash2`-colliding keys produce a nested `{k, v, n}` chain, and each additional colliding key recurses one level deeper.

I have everything I need. Here is the report.

---

# Security review — `hash_dict.ex`

## Summary
`HashDict`'s core tree operations are correct and free of injection/eval/file/command sinks; the only security-relevant property is unseeded-hash algorithmic-complexity behavior on attacker-controlled keys, and the file additionally carries a non-security correctness regression that breaks every protocol implementation.

## Security

#### `CSCAN-hash_dict-phash2-hashflood`
- **Finding ID:** `CSCAN-hash_dict-phash2-hashflood`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `hash_dict.ex:249-251` (`key_hash/1`), reached from `put/3` (`:41`), `update/4` (`:55`), `update!/3` (`:47`), `fetch/2` (`:61`), `delete/2`/`pop/3`/`dict_delete/2`.
- **Class:** Resource consumption
- **Issue:** Keys are located with `:erlang.phash2(key)`, an **unseeded, deterministic, 27-bit** hash. When many keys share one `phash2` value, the trie exhausts the hash bits and chains each additional colliding key one level deeper (the `[k | v] -> {k, v, n}` and `{k, v, n} -> do_put(n, …)` branches in `do_put`, `hash_dict.ex:129-138`). N fully-colliding keys therefore produce O(N) trie depth, making a build O(N²) and each `fetch`/`delete`/`reduce` O(N) with O(N) recursion depth. Because the hash is unseeded and public, an attacker can precompute a colliding key set offline once and reuse it against any process that inserts attacker-controlled keys into a `HashDict`.
- **Impact / threat:** An application that stores externally-supplied keys (request params, message fields, uploaded record keys) in a `HashDict` can be driven into quadratic CPU/memory blow-up by a crafted key set — a classic hash-flooding DoS. Trust boundary: the caller forwards network/file-derived keys through the documented public API (`put`/`update`/`fetch`). This is a real but marginal vector: it is **inherent to unseeded hashing and is shared by the `Map` module this file recommends as its replacement**, and weaponizing it requires an offline preimage search to gather many keys with one identical `phash2` value (≈N·2²⁷ hash evaluations), which is why it is rated Low.
- **Evidence:** `verified` for the mechanism and collision-findability; `asserted` for end-to-end large-N DoS (generating an N-way `phash2` multi-collision was too expensive to run in-sandbox).
  - Determinism/width: `elixir -e ':erlang.phash2(0)'` → `88723725` (stable across runs; 27-bit range).
  - Cheap 2-way collision via birthday search: keys `16634` and `23078` both hash to `33919375`, found within ~23k iterations.
  - Nesting mechanism, inserting those two colliding keys:
    ```
    root: {[], [], [], [], [], [], [], {16634, :a, {[], [23078 | :b], [], [], [], [], [], []}}}
    both retained: {{:ok, :a}, {:ok, :b}, 2}
    ```
    confirming each colliding key adds one `{k, v, n}` nesting level (→ O(N) depth for an N-way collision).
- **Suggested fix:** Document the constraint (keys should be trusted / low-cardinality) for this deprecated module, or — the real remedy — steer users to `Map`; there is no in-module fix short of a seeded/randomized hash, which would break the persistent structure’s stability. Practically: treat this as a "do not feed untrusted keys" caveat, matching the platform’s general stance.

### Ruled out
- **`String.to_unsafe_atom("HashDict")` — atom-table exhaustion (`:268,274,284,299,318`).** The argument is a hardcoded constant in all four call sites; it is never derived from external bytes. At most the single atom `:HashDict` is created, once. No CWE-770 exposure. *(Basis: verified — `String.to_unsafe_atom/1` is `:erlang.binary_to_atom(string, :utf8)` at `string.ex:3040`; input is a literal.)*
- **Dynamic dispatch `module.reduce/…` / `module.put/…` (`:269,275,285,302,319`).** The dispatched module is computed from the constant `"HashDict"`, not from externally-influenced data, so this is not attacker-controlled protocol/behaviour dispatch.
- **Logic bug / data loss / infinite loop in put/delete/compact.** 50 randomized operation sequences (put/delete/pop/fetch on a 40-key space, 3000 ops each) stayed fully consistent with a reference `Map` (sizes and all key lookups agreed). *(Basis: verified — differential fuzz.)*
- **Code-exec / file / command / regex / deserialization / network / crypto / template sinks.** None present in the file; it is a pure in-memory tuple-tree with `phash2`/`Bitwise` arithmetic. *(Basis: verified — full read of all 322 lines.)*
- **Inspect data-leak on protocol failure.** When `Inspect.HashDict.inspect/2` fails it falls back to dumping the raw struct map, but the intended output (`#HashDict<[…]>`) shows the same key/value contents, so no redaction is bypassed. *(Basis: verified — see failure output in caveats.)*

## Risks / caveats
- **Correctness regression, not a security finding (but the most impactful defect here).** `String.to_unsafe_atom("HashDict")` returns `:HashDict`, **not** the module atom `Elixir.HashDict`. Every protocol implementation that dispatches through it therefore raises `UndefinedFunctionError`:
  - `Enum.count/1` → `:HashDict.size/1 is undefined`
  - `Enum.to_list/1`, `Enum.reduce/3` → `:HashDict.reduce/3 is undefined`
  - `Enum.member?/2` → `:HashDict.fetch/2 is undefined`
  - `Enum.into/2` (Collectable) → `:HashDict.put/3 is undefined`
  - `inspect/1` → renders `#Inspect.Error<… :HashDict.to_list/1 is undefined …>`

  This is an **unconditional** break (it fails identically regardless of input), so it is a reliability/availability regression rather than an attacker-differential vulnerability — it does not fail *open* (it crashes, i.e. fail-closed) and would be caught by the first test that exercises a protocol. I did not classify it as a security finding for those reasons, but it should be fixed: use `String.to_unsafe_atom("Elixir.HashDict")` (with the `Elixir.` prefix, matching the intended upstream form) or reference the module directly. The core `new/put/fetch/update/delete/pop/size` API is unaffected and works correctly.
- **Unverified portion of the hash-flooding finding.** I verified `phash2` determinism, cheap 2-way collision generation, and the O(N)-depth nesting mechanism, but did not stage a full large-N multi-collision DoS in-sandbox (preimage generation cost). The parity claim "no worse than `Map`" rests on the fact that Erlang term hashing for `:maps`/`Map` is likewise unseeded; I did not independently benchmark `Map` degradation.
- **Scope.** `reduce/3` and `dict_delete/2` are `@doc false` (exported but undocumented); I treated hash-flooding as in-scope because it is reachable through the plainly-public `put/3`/`fetch/2` API, independent of those two functions.

