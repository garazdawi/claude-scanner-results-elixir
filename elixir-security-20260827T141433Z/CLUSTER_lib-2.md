# Cluster cross-file findings

#### `CSCAN-cluster-inspect-list-return-contract-mismatch`

* **Finding ID** — `CSCAN-cluster-inspect-list-return-contract-mismatch`
* **Severity** — Low
* **Confidence** — high
* **Files involved**
  * `/Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/hash_dict.ex:319` (direct call to `Inspect.List.inspect/2`, result wrapped in `concat/1`)
  * `/Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/hash_set.ex:329` (same pattern)
  * `/Users/likas/code/claude-scanner/elixir/lib/elixir/lib/inspect.ex:399-407` (`Inspect.List.inspect/2` non-empty branches return whatever `container_doc_with_opts/6` returns)
  * Out-of-cluster definitions resolved for evidence: `inspect/algebra.ex:554-577` (`container_doc_with_opts` returns a `{doc, opts}` tuple), `:365-367` (`is_doc/1` guard), `:749-751` + `:783-785` (`concat/2`/`concat/1` require `is_doc`), `:398-449` (protocol `rescue` that traps the crash).
* **Finding shape** — Other: cross-file return-contract mismatch causing a trapped crash on attacker-reachable input (a broken variant of "safe because the caller expects the old contract").
* **Evidence**
  * `hash_dict.ex:319` builds `concat(["#HashDict<", Inspect.List.inspect(module.to_list(dict), opts), ">"])`, and `hash_set.ex:329` builds `concat(["#HashSet<", Inspect.List.inspect(module.to_list(set), opts), ">"])`. Both assume `Inspect.List.inspect/2` returns a plain algebra **doc**.
  * In `inspect.ex`, for any non-empty, non-charlist list `Inspect.List.inspect/2` falls into the `keyword?` branch (399) or the `true` branch (405), both returning `container_doc_with_opts(open, term, close, opts, …)` verbatim.
  * `container_doc_with_opts/6` returns a **2-tuple** `{doc, opts}` — `{concat(left, right), inspect_opts}` for `[]` (algebra.ex:559) and `{group, inspect_opts}` for the non-empty case (algebra.ex:577). (The plain-doc variant is `container_doc/6`, which does `… |> elem(0)` at algebra.ex:498 — the hash modules do **not** call that one.)
  * `concat/1` (algebra.ex:783) folds via `concat/2` (algebra.ex:749), whose guard is `is_doc(doc1) and is_doc(doc2)`. `is_doc/1` (algebra.ex:365) accepts a tuple only when `elem(doc,0) in @docs` (a list of doc-record tag atoms, :352). For `{group, %Inspect.Opts{}}`, `elem/0` is the group doc (not an atom), so `is_doc/1` is `false` → `concat/2` raises `FunctionClauseError`. Confirmed reachable: `to_list` exists for both types (`dict.ex:104` via `use Dict`; `hash_set.ex:58`), and e.g. `HashSet.new([1,2,3])` yields `[1,2,3]`, which is neither `ascii_printable?` nor a keyword list, so it takes the tuple-returning branch.
  * The raise is **trapped**: struct inspection runs under the `try/rescue` at `algebra.ex:398-445`. With the default `safe: true` (algebra.ex:96) it degrades to an `Inspect.Error` rendering; only under `safe: false` (line 438-439) does it re-raise into the caller. Empty dicts/sets (`to_list == []` → plain doc at `inspect.ex:353`) and charlist-shaped contents render fine, so the breakage is silent for the common empty case and appears only for populated structures.
* **What the per-file pass missed** — The `hash_dict.ex`/`hash_set.ex` reviewer sees a call to `Inspect.List.inspect/2` but cannot know it now returns `{doc, opts}` without reading `inspect.ex` **and** `inspect/algebra.ex`; the `inspect.ex` reviewer sees the tuple return shape but cannot know two deprecated modules still call the implementation function directly and wrap it in `concat/1`. Only a cross-file view links the changed return contract to its two stale consumers.
* **Suggested fix** — Make the consumers tolerate the new contract. Minimal change in both files: extract the doc before concatenation, e.g. in `hash_dict.ex:319` `{doc, _opts} = Inspect.List.inspect(module.to_list(dict), opts); concat(["#HashDict<", doc, ">"])` (and symmetrically in `hash_set.ex:329`). Alternatively route through `Inspect.Algebra.to_doc/2` on the list, or expose/use a plain-doc `container_doc/6`-based helper so the `{doc, opts}` shape never reaches `concat/1`.
