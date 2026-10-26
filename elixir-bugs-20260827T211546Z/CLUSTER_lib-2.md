# Cluster cross-file findings

#### `CSCAN-cluster-inspect-list-two-tuple-return`

- **Finding ID** — `CSCAN-cluster-inspect-list-two-tuple-return`
- **Severity** — High
- **Confidence** — high
- **Files involved**
  - `lib/hash_dict.ex:316-320` (`defimpl Inspect, for: HashDict`)
  - `lib/hash_set.ex:326-330` (`defimpl Inspect, for: HashSet`)
  - `lib/inspect.ex:352-407` (`defimpl Inspect, for: List` — `Inspect.List.inspect/2`)
  - (supporting contract, out of cluster) `lib/inspect/algebra.ex:544-578` (`container_doc_with_opts/6` returns `{doc, opts}`) and `lib/inspect/algebra.ex:365-367` (`is_doc/1` guard) / `lib/inspect/algebra.ex:749-785` (`concat/1,2`)
- **Finding shape** — Type / contract mismatch across files (return-shape / two-tuple protocol contract one file emits and another cannot match)
- **Evidence**
  - `inspect.ex:399-406`: for any non-empty list, `Inspect.List.inspect/2` returns `container_doc_with_opts(...)`, whose spec/body returns a **two-tuple** `{doc, opts}` (`algebra.ex:553` `:: {t, Inspect.Opts.t()}`, body `algebra.ex:559`/`577` return `{concat(...), inspect_opts}` / `{group, inspect_opts}`). Only the empty-list clause (`inspect.ex:353`) and the charlist branch (`inspect.ex:397`) return a bare doc.
  - `hash_dict.ex:319`: `concat(["#HashDict<", Inspect.List.inspect(module.to_list(dict), opts), ">"])` — the result of `Inspect.List.inspect/2` is spliced straight into `concat/1` as if it were a doc.
  - `hash_set.ex:329`: identical pattern `concat(["#HashSet<", Inspect.List.inspect(module.to_list(set), opts), ">"])`.
  - `algebra.ex:365-367`: `is_doc/1` only admits lists, binaries, `doc_line()`, or tuples whose first element is a `@docs` atom. A `{doc, %Inspect.Opts{}}` tuple fails this guard, so `concat/2` (`algebra.ex:749`) has no matching clause and raises `FunctionClauseError` for every non-empty `HashDict`/`HashSet`. (The correct pattern is shown in `inspect.ex:114-117`, where `MapSet`'s impl destructures `{doc, opts} = to_doc_with_opts(...)` before calling `concat`.)
- **What the per-file pass missed** — Reviewing `hash_dict.ex`/`hash_set.ex` alone, `Inspect.List.inspect/2` looks like an ordinary doc-returning helper; the defect is only visible by reading the `Inspect.List` implementation in `inspect.ex` together with the `container_doc_with_opts` return spec and the `is_doc` guard in `algebra.ex`, which no single-file reviewer had in scope.
- **Suggested fix** — Destructure and thread the options in both impls, e.g.
  ```elixir
  {doc, _opts} = Inspect.List.inspect(module.to_list(dict), opts)
  concat(["#HashDict<", doc, ">"])
  ```
  (and the analogous change in `hash_set.ex`), or call `to_doc_with_opts(module.to_list(dict), opts)` and use the returned doc.
