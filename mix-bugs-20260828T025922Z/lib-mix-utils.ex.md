# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/utils.ex
<!-- sha256: 535e8b2f1e39579e397f1f8c4c3de97734a839dc3d23d47b9af2a65e2dac832f -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/utils.ex
<!-- sha256: 535e8b2f1e39579e397f1f8c4c3de97734a839dc3d23d47b9af2a65e2dac832f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (first-pass hedge "outside this file" resolved by reading `mix/tasks/deps.tree.ex`, `mix/tasks/app.tree.ex`, and executing the tree builder against both node shapes)._

## Summary

`elixir/lib/mix/lib/mix/utils.ex` is Mix's internal grab-bag of build utilities; it is largely sound. Two defects: the previously reported duplicated `to_erl_term/1` map clause (dead code, high confidence), and — newly upgraded from a first-pass hedge — a contract violation in `build_json_tree/3` where child nodes are destructured as `{name, info}` two-tuples in bypass of the documented callback, which raises `FunctionClauseError` for any node shape that is not a two-tuple (e.g. `%Mix.Dep{}`). The latter is latent today because `write_json_tree!/4` has no callers.

## Bugfixes

#### `CSCAN-utils-duplicate-map-clause`
- **Finding ID:** `CSCAN-utils-duplicate-map-clause`
- **Severity:** Low
- **Confidence:** high
- **Location:** `utils.ex:1087` (second `defp to_erl_term(map) when is_map(map)`), duplicating `utils.ex:1076`
- **Class:** Dead / unreachable code
- **Impact:** The clause at lines 1087–1096 is byte-for-byte identical to the clause at lines 1076–1085. Since the first `is_map/1` clause always matches every map, the second can never be reached. It compiles but is pure dead weight. The strongest signal here is that a copy-paste duplicate like this usually means an *intended distinct clause was lost* — e.g. a struct-specific branch — so the risk is a missing behaviour rather than a wrong one. As written it produces no incorrect output, but it is a real code defect and a maintenance hazard: anyone editing map serialization must know the second clause is inert.
- **Evidence:** verified. Re-read lines 1076–1096: the two clauses are identical. Compiling a minimal reproduction of two identical `is_map` clauses yields `warning: the following clause is redundant … previous clauses have already matched`. Prior run of `Mix.Utils.consultable(%{a: 1, b: 2})` returns a well-formed result via the first clause, confirming the second is never entered.
- **Suggested fix:** Delete the duplicate clause at lines 1087–1096. If a distinct map/struct handling was intended (e.g. rejecting or specially formatting structs), restore that instead.

#### `CSCAN-utils-build-json-tree-node-shape` — Upgraded from hedged assumption in first pass
- **Finding ID:** `CSCAN-utils-build-json-tree-node-shape`
- **Severity:** Low
- **Confidence:** low
- **Location:** `utils.ex:439` (`Enum.reduce(children, %{}, fn {name, info}, sink_map ->`), inside `build_json_tree/3`; reached via public `write_json_tree!/4` at line 424
- **Class:** `ERROR_PROPAGATES_NOT_BADMATCHES` — uncaught `FunctionClauseError`/`MatchError` from a contract mismatch (documented-return-shape / input-shape class)
- **Impact:** `write_json_tree!/4`'s documented contract (lines 411–423) says the callback "will be invoked for each node" and `node` is `term()` — i.e. child nodes are **opaque** and must be re-interpreted only by the callback. `build_json_tree/3` honours that for the *recursion* at line 445 (`build_json_tree(children, callback)`, which re-feeds each child through `callback.(node)` at line 433), but **contradicts it** at line 439, where it destructures each child directly as a `{name, info}` two-tuple without calling the callback. The same `children` list is therefore treated two incompatible ways in the same function. This only works when a node happens to be exactly a `{name, edge_info}` two-tuple. For any other node shape the `fn {name, info} ->` clause fails to match.
- **Evidence:** verified by reading callers and executing both shapes.
  - `mix/tasks/deps.tree.ex`: `--format` accepts only `pretty | plain | dot` (lines 31–42, 71–78); the task calls `write_dot_graph!` and `print_tree` only (lines 107, 124). Its callback nodes are `%Mix.Dep{}` structs / bare app atoms and children are `[%Mix.Dep{}]` (lines 138–153). There is **no `json` format and no call to `write_json_tree!`**.
  - `mix/tasks/app.tree.ex`: likewise only `dot`/`print_tree` (lines 82, 102); its nodes are `{app, type_atom}` two-tuples (lines 83, 102, 114–123).
  - Repo-wide search: `write_json_tree!`/`build_json_tree` have **zero callers and zero tests**.
  - Executed reproduction of the `build_json_tree` core against two callbacks (working Erlang/Elixir toolchain, `elixir t.exs`): an app.tree-style callback (nodes and children are `{app, atom}`) completed without error; a deps.tree-style callback (node/children are maps, as `%Mix.Dep{}` structs are) raised `RAISED: FunctionClauseError -- no function clause matching in anonymous fn/2` at exactly the `fn {name, info} ->` reducer. This confirms the line-439 destructure requires two-tuple nodes and crashes on struct/map nodes.
- **Why low confidence (not higher):** the function is entirely unused today (no caller, no test), so there is no active crash; the defect is latent and only manifests when someone wires a `--format json` (or any `write_json_tree!` caller) whose node shape is not a `{name, edge_info}` two-tuple — at which point it raises `FunctionClauseError` rather than producing JSON. I cannot exhibit a live failing call from within the shipped tasks because none invokes it.
- **Suggested fix:** interpret children through the callback rather than by direct destructure — build `sink_map` from `callback.(child)` results (`{{child_name, child_info}, _} = callback.(child)`), mirroring the recursion at line 445, so the node shape stays opaque and consistent with the documented contract.

## Risks / caveats

- `find_cycle!/1` returns the accumulated `visited` `MapSet` (unordered) rather than an ordered cycle path; this is a documentation/precision concern for how cyclic deps are displayed, not a crash, and appears to match the module's long-standing behaviour, so I did not raise it as a defect.
- I did not exercise the network path (`read_httpc/1`), the proxy-config helpers, or `symlink_or_copy` against a real filesystem/link matrix; review of those was by reading only. These are honest scope statements, not assumptions about other files.

### Ruled out

- **`build_json_tree/3` is a bug only for callers "outside this file" (first-pass hedge).** *Resolved, not ruled out* — promoted to finding `CSCAN-utils-build-json-tree-node-shape` above after reading both tree tasks and executing both node shapes. **Axes examined:** node-shape (two-tuple vs struct/map), caller existence, recursion-vs-direct-destructure consistency, empirical crash class. **Not examined:** none.
- **`last_modified_and_size/1` returns the future `mtime` after touching the file to `now` (line 334–336).** Not a bug: forcing the file to appear stale for the current run is the *intended* consequence — its recorded mtime was untrustworthy (in the future), so Mix wants to recompile it this run; returning `now` would instead make it look up-to-date and skip compilation. The `File.touch(path, now)` ensures the anomaly self-heals on the next run. Behaviour is internally consistent with the staleness comparison in `stale_stream/2`. **Axes examined:** mtime-in-future vs mtime-in-past, self-heal on next run. **Not examined:** concurrent writer racing the touch (would require a multi-process filesystem harness I did not build).
- **`escape_dot_string/2` passing a lone `\` before a non-`\`, non-`"` char through unescaped.** Verified by reading the clauses: the only correctness requirement (per the GraphViz comment) is that a trailing backslash cannot escape the closing quote, which the `<<?\\>>` clause handles by emitting `\\"`. Interior single backslashes are valid DOT and need no escaping. **Axes examined:** trailing-backslash, interior-backslash, quote-adjacent. **Not examined:** none.
- **`parse_mfa/1` doctests** (`"Foo.bar.baz"` → `:error`, `"Foo.bar/1"` → `{:ok, [Foo, :bar, 1]}`). Verified: `Mix.Utils.parse_mfa("Foo.bar/1")` returns `{:ok, [Foo, :bar, 1]}`; the multi-segment/`Foo.` cases fall through `quoted_to_m/2`'s `[]` clause and are demoted to `:error` by the `[_ | _]` guard in `with`. **Axes examined:** valid M.f/a, multi-segment, trailing-dot. **Not examined:** none.
- **`checksum/2` when `:sha512` matches / mismatches.** Verified by reading: on match the second `<-` guard (`actual != expected`) fails, `else` yields `nil`, and `Enum.find_value` returns the `{:ok, binary}` default; on mismatch it returns `{:checksum, message}`, matching the `@spec`. **Axes examined:** match, mismatch, spec shape. **Not examined:** none.
- **`consultable/1` map ordering / tuple encoding.** Verified via prior run; both map and tuple inputs produce well-formed iodata via the first matching clause. **Axes examined:** map ordering, tuple encoding. **Not examined:** none.

### Did I verify this?

- `CSCAN-utils-duplicate-map-clause` — **verified** (re-read lines 1076–1096; redundant-clause warning reproduced by compilation).
- `CSCAN-utils-build-json-tree-node-shape` — **verified as a latent contract violation** (read `deps.tree.ex`/`app.tree.ex`, confirmed no caller/test, executed both node shapes and observed `FunctionClauseError` for the struct/map shape); kept at **low confidence** because no shipped code path invokes it, so it cannot fail at runtime today.

### Dropped in re-review

None. Both prior findings are preserved; the prior "risks/caveats" hedge about `build_json_tree/3` has been upgraded to an explicit low-confidence finding rather than dropped.
