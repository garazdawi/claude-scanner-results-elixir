# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/umbrella.ex
<!-- sha256: bbd35dac52c9d7493b282dc7214387ebe120be10b666feddda9a14c6142ce96e -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/umbrella.ex
<!-- sha256: bbd35dac52c9d7493b282dc7214387ebe120be10b666feddda9a14c6142ce96e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`mix/dep/umbrella.ex` is a small internal (`@moduledoc false`) module that builds the list of umbrella-child dependencies as `Mix.Dep` structs (`unloaded/0`), loads and topologically sorts them (`loaded/0`), and memoizes the loaded list per-env in `Mix.State` (`cached/0`). In the first pass I hedged that the callee contracts were assumed; this pass reads each neighbour listed in the standing callgraph and confirms them against the actual source. The code contains no genuine, reachable behavioural bug.

## Bugfixes

No findings

## Risks / caveats

- All three functions delegate the substantive work to other modules. In the first pass this was hedged as "I assumed their documented contracts hold." I have now **verified** each contract against source:
  - `Mix.Project.apps_paths/1` (`project.ex:468-479`): returns `nil` when `config[:apps_path]` is falsy; otherwise returns a map built by `to_apps_paths/2` (a `for … into: %{}` comprehension). Never returns a bare list. The `if apps_paths = …` / `else []` guard in `unloaded/0` handles both the `nil` and the (truthy, possibly empty `%{}`) map cases correctly — an empty map iterates to `[]`.
  - `Mix.State.read_cache/2` (`state.ex:48-50`): `:persistent_term.get({__MODULE__, key}, default)` with `default \\ nil` — returns `nil` on miss, stored term on hit. `Mix.State.write_cache/2` (`state.ex:52-55`): stores and **returns the value**. Confirms the `read_cache(key) || write_cache(key, loaded())` idiom returns the freshly-cached list on a miss.
  - `Mix.Dep.Loader.load/3` (`loader.ex:84`) and `validate_manifest/1` (`loader.ex:423`): both take and return a `%Mix.Dep{}` struct — the `Enum.map` pipeline in `loaded/0` is well-typed.
  - `Mix.Dep.Converger.topological_sort/1` (`converger.ex:14`): takes a list of `%Mix.Dep{}` and returns a list of them; it may `Mix.raise` on a self-dependency but that is intended behaviour, not a crash this file introduces.
  - Confidence: N/A (no defect; verification note).
- I did not execute a live umbrella build. The logic in this file contains no arithmetic, parsing, or state-threading whose outcome a reasoned read (now backed by the verified callee contracts above) leaves in doubt.

### Ruled out

- **`cached/0` `read_cache(key) || write_cache(key, loaded())` truthiness** — considered whether caching an empty result would defeat memoization. `loaded()` always returns a list; `[]` is truthy in Elixir (only `nil`/`false` are falsy), and `read_cache/1` returns `nil` only on a genuine miss (verified `state.ex:48-50`). So a cached `[]` is returned without recomputation, and `write_cache/2` returns the value it stored (verified `state.ex:52-55`). The `||` short-circuit is correct; `loaded()` is not evaluated on a hit. Not a bug.
- **`unloaded/0` `path: path` vs `dest: dest_path`** — verified against `Mix.SCM.Path` (`scm/path.ex:16,27-28,48-49,59,69,79`): the module consumes `opts[:path]` (raw/relative source) and `opts[:dest]` (expanded absolute destination) as distinct fields, and `checkout`/`checked_out?`/`equal?`/`format` key off `:dest`. Setting both, with `dest` = `Path.expand(path)`, matches exactly how the SCM reads them. Intentional, not a mismatch.
- **`loaded/0` filter `Mix.Dep.available?(dep) and dep.app in apps`** — both operands are booleans (`available?/1` returns a boolean, `in` returns a boolean), so `and` (which requires boolean operands) is safe; no `&&`-vs-`and` truthiness hazard. Not a bug.
- **`for {app, path} <- apps_paths`** — `apps_paths/1` returns a map (iterated as `{key, value}` pairs) or `nil` (verified `project.ex:468-479`); the `if apps_paths = …` guard handles `nil` with `[]`, and a map comprehension over `{app, path}` pairs is well-formed. No missing-clause/`nil` crash.
- **Double `Enum.map` pipeline in `loaded/0`** — the first map loads each dep and rewrites its `.deps`; the piped second map applies `validate_manifest/1`. Both consume and produce `%Mix.Dep{}` (verified `loader.ex:84,423`) and traverse the same list; the accumulator is a plain list rebind with no dropped element. Not a state-threading defect.

### Did I verify this?

Yes, for the callee contracts. This pass read the four neighbour modules named in the standing callgraph and cited exact line ranges: `mix/project.ex:468-497` (`apps_paths/1`, `to_apps_paths/2`), `mix/state.ex:48-55` (`read_cache`/`write_cache`), `mix/dep/loader.ex:84,423` (`load/3`, `validate_manifest/1`), `mix/dep/converger.ex:14-34` (`topological_sort/1`), and `mix/scm/path.ex:16-79` (`:path`/`:dest` consumption). Every first-pass "I assumed" is now grounded in read source. I did not run a live umbrella build; the ruled-out entries are pure Elixir-evaluation-semantics dismissals with no differential/second-implementation axis to execute, so no sandbox run would change their verdict.

### Dropped in re-review

No findings dropped — the first pass reported none. The three hedge phrases flagged ("I assumed", "out of scope for this file", "I did not execute") were resolved by reading the neighbours rather than by upgrading to findings, because every verified contract matches what this file relies on; none revealed a disagreement that would justify a low-confidence finding.
