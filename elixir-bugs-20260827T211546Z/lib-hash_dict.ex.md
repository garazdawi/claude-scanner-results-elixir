# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/hash_dict.ex
<!-- sha256: 773c7fc386a615c78e7f7a82a3dbc01378551aade240f3e255ae6949152d4dee -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:12:13Z_

## Summary

The four protocol implementations for `HashDict` (`Enumerable`, `Collectable`, `Inspect`) resolve the target module with `String.to_unsafe_atom("HashDict")`, which yields the atom `:HashDict` instead of the real module atom `:"Elixir.HashDict"`, so every protocol-driven operation on a `HashDict` crashes with `UndefinedFunctionError`.

## Bugfixes

#### `CSCAN-hash_dict-protocol-module-atom-missing-elixir-prefix`
- **Finding ID** — `CSCAN-hash_dict-protocol-module-atom-missing-elixir-prefix`
- **Severity** — High
- **Confidence** — high
- **Location** — `hash_dict.ex:268` (`Enumerable.reduce/3`), `:274` (`Enumerable.member?/2`), `:284` (`Enumerable.count/1`), `:299` (`Collectable.into/1`), `:318` (`Inspect.inspect/2`)
- **Issue** — Each impl computes `module = String.to_unsafe_atom("HashDict")` to name the module it delegates to (the "avoid deprecation warning" trick). Elixir module atoms are prefixed with `Elixir.`, so the `HashDict` module is the atom `:"Elixir.HashDict"`. `String.to_unsafe_atom("HashDict")` produces the *unprefixed* atom `:HashDict`, which is not a loaded module. Every subsequent `module.reduce/put/fetch/size/to_list` call therefore dispatches to a non-existent module. The correct source (matching upstream) is `String.to_atom("Elixir.HashDict")`.
- **Class** — Pattern match on the wrong shape / Contract violation (wrong module atom → protocol contract unmet).
- **Impact** — Every documented public path that goes through these protocols is broken: `Enum.*` over a `HashDict` (`Enum.to_list/1`, `Enum.map/2`, `Enum.member?/2`, `Enum.count/1`, `Stream` consumers), building one via `Enum.into/2` / `for ... into: HashDict.new()` (`Collectable`), and `inspect/1` / `IO.inspect/1` of a `HashDict`. All raise `UndefinedFunctionError (module :HashDict is not available)` at runtime. The direct `HashDict.*` API still works, but the entire protocol surface is unusable.
- **Evidence** — `verified`.
  Atom produced:
  ```
  $ elixir -e 'IO.inspect(String.to_unsafe_atom("HashDict")); IO.inspect(HashDict == String.to_atom("Elixir.HashDict"))'
  :HashDict
  true
  ```
  End-to-end failure through the public `Collectable`/`Inspect`/`Enumerable` paths:
  ```
  $ elixir -e 'd = Enum.into([a: 1, b: 2], HashDict.new); IO.inspect(Enum.to_list(d))'
  ** (UndefinedFunctionError) function :HashDict.put/3 is undefined (module :HashDict is not available)
  ...
  function :HashDict.to_list/1 is undefined (module :HashDict is not available)
  ```
  For contrast, the direct API works: `HashDict.new |> HashDict.put(:a,1) |> HashDict.fetch(:a)` returns `{:ok, 1}`.
- **Suggested fix** — In all five call sites replace `String.to_unsafe_atom("HashDict")` with the fully-qualified module atom, e.g. `String.to_atom("Elixir.HashDict")` (or `String.to_unsafe_atom("Elixir.HashDict")`).

## Risks / caveats
- I verified against the locally built Elixir at `/opt/elixir`, which reproduces the bug exactly, confirming the reviewed source is what is compiled and shipped.
- The core direct API (`new`, `put`, `update`, `fetch`, `delete`, `pop`, `size`, `reduce/3`) was exercised and behaves correctly; the defect is confined to the three `defimpl` blocks. `Enumerable.slice/1` returning `{:error, __MODULE__}` (→ `Enumerable.HashDict`) is the correct convention and is not a bug.

### Ruled out
- **`do_delete` node-compaction (`do_compact_node`, `@node_template` match at line 183)** — considered whether deletion could corrupt the trie or drop entries; drove multi-key insert/delete sequences through the public API (`put`/`delete`/`size`/`fetch`) and observed correct sizes and lookups. Basis: `verified`.
- **`slice/1` returning `{:error, __MODULE__}`** — `__MODULE__` inside `defimpl Enumerable, for: HashDict` is `Enumerable.HashDict`, which is the documented signal that the enumerable has no fast slicing and Enumerable should fall back to reduce; not a defect. Basis: `asserted` (standard `Enumerable` contract).
- **`key_hash`/`key_mask`/`key_shift` bit arithmetic** — `@node_bitmap 0b111`, `@node_shift 3`, `@node_size 8` are internally consistent (mask selects one of 8 slots, shift consumes 3 bits per level); exercised indirectly by the verified core-op round-trips above. Basis: `verified`.

