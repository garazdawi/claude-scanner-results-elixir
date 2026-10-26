# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/converger.ex
<!-- sha256: 25cf3c6294021362afd72dd90c0c5bf7287f02aba6df4a0eef67412564879ae8 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:13:06Z_

The file under review is byte-identical to the copy shipped in the built-from-source Elixir 1.21-dev at `/opt/elixir` (the same tree the sandbox's `elixir`/`mix` were built from). It is verbatim upstream Mix converger code. I still worked through the candidate defects individually.

## Summary

`Mix.Dep.Converger` implements Mix's breadth-first dependency convergence and topological sort; after enumerating every candidate defect class against its three documented functions (`topological_sort/1`, `converge/1`, `converge/4`) and their private helpers, I found no genuine reachable bug — the code matches the tested upstream and its state-threading, `cond`/`with` fallthroughs, and manager/only/targets merging are all total for the shapes callers produce.

## Bugfixes

No findings

## Risks / caveats

- **Basis of the "no findings" conclusion.** The reviewed file is identical (`diff` → `IDENTICAL`) to `/opt/elixir/lib/mix/lib/mix/dep/converger.ex`, which is what the sandbox `elixir`/`mix` were compiled from and against which `converger_test.exs` passes. That gives strong evidence the observable behaviour is the intended, tested behaviour rather than an injected defect.
- **Items considered and ruled out** (below). Reachability of the deeper `all/9` recursion depends on private state threaded by `converge/4`; I traced it by reading, not by exhaustively driving every convergence/divergence/optional-add-back permutation through a live project, so a subtle cross-file invariant break (e.g. in `Mix.Dep.Loader`) would not be caught from this file alone.

### Ruled out

- **`topological_sort/1` — `:digraph.add_edge(graph, other_app, app)` with a non-vertex `other_app`.** `add_edge/3` returns `{:error, {:bad_vertex, _}}` (it does not raise) and the result is discarded. This is not reachable to harmful effect: every child in a dep's `.deps` is also present as a top-level entry in the flat `deps` list passed in (non-fulfilled optionals are stripped by `reject_non_fulfilled_optional_and_validate_app/1` before sort), so all edge endpoints are vertices. No crash and no missing edge in practice; matches upstream.
- **`Enum.find/2` in the topsort result mapping returning `nil`.** `:digraph_utils.topsort/1` returns exactly the vertices, and every vertex was added from a `deps` element, so `Enum.find` always matches. Not reachable.
- **`String.to_unsafe_atom("diverged#{key}")` in `with_matching/5` — atom-table growth.** `key` is only ever the literals `:only` or `:targets` (the only two callers in `with_matching_only_and_targets/3`), so the produced atoms are a fixed two-element set, not attacker-controlled. Not atom exhaustion.
- **`req_mismatch/2`'s `with ... else _ -> nil`.** When `status` is not `{:ok, vsn}` or the version matches, it returns `nil`, and the `cond` guard `vsn = req_mismatch(...)` correctly treats `nil` as falsy; a real version string is always truthy. Behaviour is correct and total.
- **`converge?/2` using `and`.** Every operand (`==` comparisons, `opts_equal?/2` via `Enum.all?`, and the `Mix.SCM.equal?/2` callback contract) yields a boolean, so strict `and` never raises `BadBooleanError`. Correct.
- **`sort_manager/3` (in_upper false) priority selection.** `@managers -- (@managers -- (wrap(a) ++ wrap(b)))` reduces to the managers actually present, preserving `[:mix, :rebar3, :make]` priority order, and `List.first(...) || other_manager || manager` handles the both-nil case. No wrong-manager selection.
- **`all/9` accumulator/breadth threading.** The `:nomatch` branch recurses over remaining siblings first (`upper`, `breadths` unchanged) then over children (`upper := breadths`, `breadths := new_breadths`), and re-adds `no_longer_optional` deps to the worklist — consistent with the documented BFS/override algorithm; no stale binding or dropped field observed.

