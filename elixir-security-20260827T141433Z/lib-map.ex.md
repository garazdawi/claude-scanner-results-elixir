# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/map.ex
<!-- sha256: cfae2326e8c87ad0fe10400dec3eae7e3c7bd90ebbde85d7d51a3c71af02387f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:40:56Z_

I have completed a thorough review. Elixir works from source, and I've verified each candidate behavior by running it. Here is the report.

---

# Security review — `lib/elixir/lib/map.ex`

## Summary
`Map` is a pure functional key-value module that delegates almost entirely to Erlang's `:maps`; it has no code-execution, atom-creation, file, network, deserialization, shared-state, or crypto sinks reachable through its documented public API, and I found no meaningful security issues.

## Security

`No findings`

### Ruled out

- **`from_struct/1` dispatches on a caller-supplied atom (`struct.__struct__()`, lines 1085–1088).** This is a deprecated path (`atom | struct` per `@spec`) that does a 0-arity call on a module the *caller* names; there is no argument injection and no external-bytes→atom conversion in this file (the atom-table-exhaustion precondition is absent). To be an attack, untrusted data would have to already be an atom that the caller forwards — that conversion, not this call, would be the vulnerability. **Basis: verified** — `Map.from_struct(:erlang)` and `Map.from_struct(NoSuchModuleXYZ)` both raise `UndefinedFunctionError` (0-arity, no side channel); command + output shown above.
- **`inspect` inside `raise` messages (lines 1020, 1063) and `KeyError ... term: map` (line 790).** The interpolated value in `get_and_update/3`/`get_and_update!/3` is the return value of the caller's own `fun`, and the `KeyError` term is the caller's own map — neither crosses a trust boundary into a downstream interpreter, and `inspect` produces an inert string (no template/query/shell sink). Not template injection. **Basis: verified** by reading the two `raise` sites and the `KeyError` construction.
- **`equal?/2` uses `===/2` (line 1120), which is not constant-time.** It is documented as a completeness convenience for structural map equality, not a secret/MAC comparison primitive; nothing here claims a security property. Constant-time comparison (`:crypto.hash_equals/2`) is not the contract of this function. **Basis: verified** by reading the doc and body.
- **`badmap` catch-all clauses (`other -> :erlang.error({:badmap, other})`, e.g. lines 399, 426, 514, 596, 628, 739; and `take/drop/split` `non_map` clauses).** These are *live* error paths, not dead defensive code — the `%{}` pattern matches any map, so `other`/`non_map` matches only non-maps and is reachable from the public API. **Basis: verified** — `Map.get("notamap", :k)` raises `BadMapError`; output shown above.
- **`Map.new/1` on a malformed list → `:maps.from_list` (line 222); `Map.new/2` / `take` / `drop` / `split` deprecation paths call `Enum.to_list` (lines 229, 543, 846, 886).** A non-pair list or an infinite stream causes an `ArgumentError` or a non-terminating fold, but the enumerable is the caller's own data (no trust boundary), and the failure is a self-inflicted crash/hang, not memory-safety or injection. **Basis: verified** — `Map.new([1,2,3])` raises `ArgumentError`; output shown above.
- **Non-tail-recursive helpers `do_map`/`do_filter`/`do_reject`/`do_split_with` (lines 261, 1161, 1198, 943) — resource consumption.** These build `[h | recurse()]` over map entries, but the BEAM grows the process stack on the heap; cost stays O(n) in the map the caller already holds (no amplification), with no stack crash. **Basis: verified** — `Map.new/2` and `Map.filter/2` over a 2,000,000-entry map complete in ~1 s / ~0.75 s linearly with no crash; output shown above.
- **No occurrences** of `Code.eval*`/`compile*`, `Module.create`/`concat`, `String.to_atom`/`binary_to_atom`, `binary_to_term`, `System.cmd`/`shell`/`:os.cmd`, `Regex.compile`/`~r`, `Path.*`/`File.*`, `:erl_tar`/`:zip`, `EEx.*`, `Base.decode*`, `URI.parse`, `:crypto`, `:ets`/`:persistent_term`/`Application.put_env`/`Process.put`, or compile-time FS/network reads. **Basis: verified** by full read of all 1249 lines; the only compile directive is `@compile {:inline, ...}` (line 131).

## Risks / caveats
- The folder threat model and Unicode/tokenizer obligation checklist supplied as standing context govern `elixir/lib/elixir/src` (the compiler front-end); they do not apply to `map.ex`, which performs no tokenizing, normalization, or identifier handling. I scoped the review to this file's own public API accordingly.
- User-supplied `fun` callbacks (`filter/2`, `update/4`, `merge/3`, `get_and_update/3`, etc.) run arbitrary caller code, and functions accepting `Enumerable.t()` can be handed infinite/expensive enumerables — both are inherent to the documented contract and are the caller's own code/data, not an external trust boundary this module can or should defend.
- Findings that would require external bytes to first be converted into an atom, module, or AST elsewhere and then forwarded into `from_struct/1` are out of scope here (the conversion is the vulnerability, and it does not occur in this file); I verified only that the sink itself performs no injection.

