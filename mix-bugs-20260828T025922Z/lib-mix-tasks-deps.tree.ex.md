# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.tree.ex
<!-- sha256: 2d66bff32faf62a2e2395b8d402060c2c60b7036dc0e3487eccd1be5d7a2d643 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:49:43Z_

## Summary

`mix deps.tree` prints the project dependency tree; the task logic is sound, but the `@moduledoc` documents and demonstrates an `--include` option that the task does not implement.

## Bugfixes

#### `CSCAN-deps.tree-include-option-unimplemented`
- **Finding ID** — `CSCAN-deps.tree-include-option-unimplemented`
- **Severity** — Low
- **Confidence** — high
- **Location** — `deps.tree.ex:65-67` (`@moduledoc` example) and `deps.tree.ex:71-78` (`@switches`)
- **Class** — Documentation mismatch
- **Impact** — A user who follows the published example `mix deps.tree --include bandit --include phoenix` expecting the tree to be filtered down to just those two dependencies gets the **entire** tree instead, with no error and no warning. `--include` is neither listed in the "Command line options" section nor present in `@switches`, so it is silently discarded. The documentation promises a filtering feature that does not exist.
- **Evidence** — `verified`. `@switches` (lines 71-78) has no `include` key, and no code path consumes an `:include` option. Running the parser the task uses:
  ```
  $ elixir -e 'IO.inspect(OptionParser.parse(["--include","bandit","--include","phoenix"], switches: [only: :string, target: :string, exclude: :keep, umbrella_only: :boolean, format: :string, output: :string]))'
  {[include: "phoenix"], [], []}
  ```
  The two `--include` values collapse to `[include: "phoenix"]` (not `:keep`, so last wins) and, crucially, `args` comes back `[]`. In `run/1` the empty-`args` branch (line 94) is taken, so `root` is the whole project app and the full tree is printed — the `--include` values never influence the output. The doc's options list (lines 20-53) also never mentions `--include`, confirming the example is the defect rather than a missing switch entry.
- **Suggested fix** — Remove the `--include` example block (lines 65-67) from the `@moduledoc`, since no include feature exists; the intended filtering is already covered by `--exclude`. (Alternatively, if include-filtering is actually wanted, implement it — add `include: :keep` to `@switches` and filter `select_and_sort/3` to keep only listed apps — but that is a feature, not a doc fix.)

## Risks / caveats

- I reviewed only this file. The behaviour of `Mix.Dep.Converger.converge/1`, `Mix.Utils.print_tree/3`, and `Mix.Utils.write_dot_graph!/5` was treated as a trusted boundary; a defect inside those helpers would not be visible here.
- The callback closure's `find_dep(deps, app).deps` (line 147) assumes `converge/1` returns exactly one dep per app and that every `%Mix.Dep{}` handed to the callback is present in `deps`. Both hold under the converger's documented dedup/flatten contract, so I did not treat it as a finding; if that invariant were ever broken it would raise on a `nil.deps`, but that requires violating the converger contract, which is out of scope.
- `String.to_unsafe_atom/1` on `--only`/`--target`/`--exclude`/positional app names (lines 88, 99, 130) grows the atom table from CLI input, but these are operator-controlled arguments to a build tool invocation, not untrusted network/file input, so this is expected behaviour rather than a bug.

