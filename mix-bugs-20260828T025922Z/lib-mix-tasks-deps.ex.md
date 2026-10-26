# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.ex
<!-- sha256: 6ba99102f68809933765c4dde8ca9dfc7a2d9e61c096cb514c06252b9410459c -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:47:35Z_

## Summary

`elixir/lib/mix/lib/mix/tasks/deps.ex` is the `mix deps` task (`run/1` is the only public entry point); it converges deps, optionally filters by app-name arguments, and prints status. The logic is small and largely correct — the only defect of note is the deliberate-but-unbounded creation of atoms from raw CLI arguments.

## Bugfixes

#### `CSCAN-deps-unsafe-atom-from-args`
- **Finding ID:** `CSCAN-deps-unsafe-atom-from-args`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `deps.ex:209`, `run/1` (`Enum.map(&String.to_unsafe_atom/1)`)
- **Class:** Unclassified (atom-table growth / ATOM_TABLE_NOT_EXHAUSTED)
- **Impact:** Every dependency name passed on the command line is converted with `String.to_unsafe_atom/1`, which *creates* a brand-new atom whenever the name has never been seen before. A typo (`mix deps phoeenix`) permanently adds an atom for the lifetime of the VM, and a caller/script that shells out `mix deps <name>` with a large number of distinct, arbitrary names in a single invocation can grow the atom table without bound (the atom table is never garbage-collected and defaults to ~1M entries, after which the VM aborts). Because each `mix` invocation is a fresh OS process, this is only exploitable within a single command line, so real-world impact is limited — but the safe alternative (`String.to_existing_atom/1` guarded with a rescue) would both avoid the growth and still let genuine unknown names fall through to the existing "unknown dependency" path.
- **Evidence:** verified — `elixir -e 'IO.inspect(String.to_unsafe_atom("some_random_atom_xyz_123"))'` printed `:some_random_atom_xyz_123`, confirming a non-existing string is materialized into a new atom. The mapped values feed `filter/2`, whose only consumers are `==` comparisons and the `apps -- ...` unknown computation, so a non-existing atom is never required to already exist — meaning `to_existing_atom` (with a rescue producing an "unknown" name) would be functionally equivalent for legitimate deps while removing the growth.
- **Suggested fix:** Convert with `String.to_existing_atom/1` inside a `try`/`rescue ArgumentError`, treating names that fail conversion as unknown deps (they cannot match any converged dep anyway), instead of `String.to_unsafe_atom/1`.

## Risks / caveats

- I did not spin up a full Mix project to exercise `run/1` end to end (it requires `Mix.Project.get!/0` and `Mix.Dep.Converger.converge/1`); the control-flow reasoning is from reading the source plus verifying `String.to_unsafe_atom/1` behaviour directly in `iex`.
- Behavioural quirk, not classified as a bug: without `--all`, `converge/1` only returns deps for the current `Mix.env()`/`Mix.target()`, so `mix deps some_test_only_dep` in `:dev` reports `warning: unknown dependency some_test_only_dep` even though the dep is declared. This appears intentional (the `--all` flag is the documented escape hatch) and the docs do not promise otherwise, so I did not file it.
- `check_lock/1`, `format_dep/1`, `format_status/1`, and each SCM's `format_lock/1` live in other files (`Mix.Dep`, `Mix.SCM.*`) and were out of scope; I confirmed only that `run/1` calls them with the shapes they expect (`%Mix.Dep{}` and `dep.opts`).

### Ruled out
- **`filter/2` crashing / mis-reporting order** — `Enum.find` returning `nil` is absorbed by `List.wrap/1` (→ `[]`), the `--` computes unknowns correctly against the uniq'd input, and selected deps preserve argument order as documented ("since v1.20.0"). Verified by reading the data flow; no unhandled shape.
- **Pattern match `%Mix.Dep{scm: scm, manager: manager} = dep`** — `deps` is always a list of `%Mix.Dep{}` structs from `converge/1`; the match is total for that input.
- **`--all` combined with app filters** (`mix deps --all phoenix`) — `opts[:all]` only affects `loaded_opts` for `converge/1`; the presence of `apps` still routes to `filter/2`. This yields "show phoenix regardless of env", a coherent result, not a defect.

