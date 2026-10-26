# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/access.ex
<!-- sha256: 718a27fb1b28b94563b08770270076b20adfa563a09ff8c4784d9a79c3551096 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:32:45Z_

All documented paths behave correctly and match the reference build. This is clean, genuine Elixir `Access` source with no planted or latent defect reachable through its public API.

## Summary
A full behavioral differential (on-disk file vs. reference build), all 56 doctests, and targeted fuzzing of every accessor found no genuine, reachable bug in `access.ex`.

## Bugfixes

No findings

### Ruled out

- **`values/0` map ordering looked divergent (`[3,2,1,4]` vs `[3,1,2,4]`) between two runs** — Not a bug. The divergence is Erlang small-map iteration-order nondeterminism *across separate VM instances*, not a code difference. Within any single run, `get_in(m, [Access.values()])` and the "gets" of `get_and_update_in` are byte-identical, and the ordered keyword-list case matched exactly. Basis: `verified` — ran `order.exs` five times; raw `Enum.map` iteration order itself flips between `[3,1,2,4]` and `[3,2,1,4]`, and `get == gau` printed `true` every run.
- **`slice/1` `:get` (via `Enum.slice`) vs `:get_and_update` (manual `normalize_range` + `get_and_update_slice`) could select different indices** — Ruled out. Basis: `verified` — fuzzed every `Range.new(first,last,step)` for `first,last ∈ -8..8`, `step ∈ 1..3`, list length `0..6`; the selected-element sets were identical in all cases (`done`, zero `DIVERGE` lines), including negative-index and out-of-range endpoints.
- **`at/1` negative out-of-bounds in `:get_and_update` returning a truncated list from the `{default_fun.(), list}` branch** — Ruled out; that branch is only reached at top level with `updates == []`, so `list` is the full original. Basis: `verified` — `at(-9)` gau/pop both returned `{nil, [1,2,3,4,5,6]}`, matching the reference build.
- **`fetch/2` on a non-keyword list with an atom key (`:lists.keyfind` over non-tuple elements)** — Ruled out; `keyfind` skips non-tuples rather than raising. Basis: `verified` — `Access.fetch([1,2,3], :a)` returned `:error`.
- **Struct `fetch`/`get`/`get_and_update`/`pop` misattributing `UndefinedFunctionError` via `raise_undefined_behaviour`** — Ruled out; the macro guards on a specific `{^module, :fetch, [^container, ^key], _}` stacktrace head and reraises the original otherwise. Basis: `asserted` (guard is exact-match on MFA) plus `verified` that the documented "does not implement the Access behaviour" path and all non-container error messages (`Access.key/2 expected...`, `could not put/update key ... on a nil value`, `Access.at!/1`, list-int-key `ArgumentError`) reproduce as documented.
- **`key/2`, `key!/1`, `elem/1`, `find/1`, `filter/1` guard/arity mismatches** (e.g. `filter` accepting any-arity vs `find` requiring arity-1; `elem` rejecting negative index) — Ruled out as bugs: each rejection matches the `@spec`/`@doc` contract, and every documented example plus `:pop`/update/negative-index variant reproduced identically to the reference build. Basis: `verified` via the `battery.exs`/`edge.exs` runs and the passing doctest suite (`56 passed`).

## Risks / caveats
- I reviewed only `access.ex`; behavior depends on `Map`, `Keyword`, `Enum`, and `Range`, which I treated as correct oracles rather than re-auditing.
- The differential oracle assumes the installed `/opt/elixir` `.beam` and the on-disk source are the same intended code; they agreed everywhere tested, which is itself the evidence of no regression.
- Map-keyed `values/0`/`key/2` results inherit Erlang's unspecified map iteration order; callers relying on a stable order across nodes/VM restarts would be relying on undocumented behavior, but that is a property of Erlang maps, not a defect in this file.

