# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/loadpaths.ex
<!-- sha256: fe254fe901e4501180f0cb1841320969963a996c2c28607f02b037b7267e8f93 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/loadpaths.ex
<!-- sha256: fe254fe901e4501180f0cb1841320969963a996c2c28607f02b037b7267e8f93 -->

_Re-analysed on 2026-08-27_

The file has exactly these operations of interest:
- `run/1` → `Mix.Task.run("deps.loadpaths", args)` — fixed task name, `args` forwarded (CLI/operator-controlled).
- `load_project/1` → `File.rm_rf(app_path)`, `File.rm_rf(consolidation_path)`, `Code.prepend_path(compile_path)`.
- `Mix.shell().info("... #{inspect(config[:app])} ...")` gated by `Mix.debug?()`.

In this pass I opened the neighbours that the first pass had only traced by assumption:
- `Mix.Project.build_path/1`, `app_path/1`, `compile_path/1`, `consolidation_path/1`, `manifest_path/1` (`lib/mix/lib/mix/project.ex:769–917`).
- `Mix.Dep.ElixirSCM.read/1` and `update/1` (`lib/mix/lib/mix/dep/elixir_scm.ex:16–40`).

## Summary
`Mix.Tasks.Loadpaths` only manipulates paths and code-load-paths derived from the operator's own `mix.exs`/environment config, so no untrusted (dependency/archive/network) data reaches any sink in this file. Confirmed by reading the path-builder and manifest-reader neighbours: every sink argument is built from operator-controlled config keys and environment variables, and the manifest read is scoped to the top project's own `_build` tree.

## Security
No findings

### Ruled out

- **`File.rm_rf(Mix.Project.app_path(config))` / `consolidation_path(config)` — path traversal / arbitrary deletion.** Confidence: High. Verified by reading `project.ex:853–917`: `app_path` returns `config[:deps_app_path] || Path.join([build_path(config), "lib", Atom.to_string(config[:app])])`; `consolidation_path` returns `config[:consolidation_path] || Path.join(app_path|build_path, "consolidated")`. `build_path` (`project.ex:769–786`) resolves only from `config[:deps_build_path]`, `MIX_BUILD_PATH`, `MIX_BUILD_ROOT`, `config[:build_path]`, or the `"_build"` default. Every input is an operator-controlled `mix.exs` config key or environment variable. In `loadpaths.ex` `config` is `Mix.Project.config()` (top project, line 36), not a dependency's config, so no dependency/archive/network **data** feeds any of these. A value that could redirect the delete is already RCE-by-design via `mix.exs`.
  - Axes examined: source of every path component (config keys, env vars, defaults); which project's config is passed. | Not examined: none.

- **`Code.prepend_path(Mix.Project.compile_path(config))` — load-path / code-execution.** Confidence: High. Verified: `compile_path` = `Path.join(app_path(config), "ebin")` (`project.ex:887–889`), i.e. the project's own `_build/<target><env>/lib/<app>/ebin`, derived entirely from operator config as above. No external data.
  - Axes examined: derivation of the prepended directory. | Not examined: none.

- **`Mix.Dep.ElixirSCM.read()` gating the deletion.** Confidence: High. Verified by reading `elixir_scm.ex:27–40`: `read/1` reads `Path.join(Mix.Project.manifest_path(), "compile.elixir_scm")`. `manifest_path` (`project.ex:827–837`) resolves to `<app_path>/.mix` under the top project's own `build_path` — the project's own `_build` tree, not a dependency-writable location without pre-existing RCE. The return is `{:ok, old_vsn, old_scm, _, _}`; `loadpaths.ex` uses only `old_vsn`/`old_scm` in an equality test (line 62) that decides *whether* to force a recompile — never a path. A tampered manifest can at most force a self-inflicted rebuild inside the project's own `_build`. Malformed/undecodable content is caught by the `rescue` (returns a benign default tuple) and further by the `_ -> :ok` catch-all in `loadpaths.ex`.
  - Axes examined: what the return value controls (gate vs. path); source location of the manifest file. | Not examined: none.

- **`:erlang.binary_to_term/1` (without `:safe`) inside `Mix.Dep.ElixirSCM.read/1` — CWE-502 / CWE-770 atom exhaustion.** Confidence: Medium. This sink is not in the file under review; it lives in `elixir_scm.ex:31`. I read it to resolve the first pass's hedge: the input is the top project's own `_build/.../.mix/compile.elixir_scm`, written by `update/1` via `term_to_binary` (`elixir_scm.ex:16–24`). Decode errors are caught by the surrounding `try/rescue`, but atom creation is a decode-time side effect not stopped by the rescue. Reaching it with a hostile payload requires an entity that can already write into the project's own `_build` — which for a malicious dependency build script is RCE-by-design, and for the operator is out of the attacker model. It belongs to `elixir_scm.ex`'s own review (where the trust of the `_build` manifest against a malicious-dependency *writer* should be judged); noted here only because `load_project/1` is a caller.
  - Axes examined: source of the deserialized bytes; whether the caller passes an alternate (dependency) manifest_path (it does not — default arg = top project). | Not examined: none reachable from this file.

- **`Mix.shell().info("...#{inspect(config[:app])}...")` — format/log injection.** Confidence: High. Runs only under `Mix.debug?()` (line 63); the interpolated value is `config[:app]`, an operator-controlled atom rendered through `inspect` (escaped). No untrusted data, no `~`-directive interpretation (the string is passed as data to `info/1`, not as a format string).
  - Axes examined: source of interpolated value; escaping. | Not examined: none.

- **`Mix.Task.run("deps.loadpaths", args)` — dynamic dispatch / arg injection.** Confidence: High. Task name is a string literal; `args` are operator CLI arguments forwarded verbatim with no sink in this file. The `deps.loadpaths` task's handling of `args` is in scope for that task's own review, not this file.
  - Axes examined: dispatch target (literal); presence of any local sink for `args` (none). | Not examined: none in this file.

- **`if "--from-mix-deps-compile" not in args` / `if config[:app]` / `case … _ -> :ok`.** Confidence: High. Ordinary control flow (recursion guard, umbrella guard, catch-all), not security gates — not dead defensive code.
  - Axes examined: role of each branch. | Not examined: none.

### Did I verify this
- I read the file under review and both callgraph neighbours it reaches (`Mix.Project` path builders at `project.ex:769–917`; `Mix.Dep.ElixirSCM.read/update` at `elixir_scm.ex:16–40`). The first pass's caveats ("assessed by reading, not execution") are now discharged: reading the neighbours *confirmed* that every `rm_rf`/`prepend_path`/`manifest` argument is built solely from operator config keys, env vars, and defaults, and that `read/1` targets the top project's own manifest path.
- No positive vulnerability exists in this file, so there is nothing to reproduce at runtime; the conclusions rest on the source trace above, not on differential execution. There are no two-implementation conformance axes in play here, so no "Not examined" axis required an execution trial — each ruled-out entry's axes are the data-source/derivation questions the reads answered directly.

### Dropped in re-review
None. All prior ruled-out entries are preserved. The three prior "Risks / caveats" hedges were not findings; they were assumptions, and all three are now folded into the ruled-out entries above as verified statements:
1. "Helpers assessed by reading, not execution" → resolved: the reads confirm operator-only derivation of every path (see the `app_path`/`compile_path`/`consolidation_path` entries).
2. "`binary_to_term` without `:safe` reachable from `load_project/1`" → resolved to a Medium cross-file note scoped to `elixir_scm.ex`, with the manifest source verified as the top project's own `_build` (not a dep-writable path via the default arg).
3. "Did not audit `deps.loadpaths`" → unchanged and correct: that delegated task owns its `args` handling; no sink for `args` exists in this file.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`File.rm_rf(Mix.Project.app_path(config))` / `consolidation_path` — path traversal / arbitrary deletion.** The path is `Path.join([build_path, "lib", Atom.to_string(config[:app])])`, where `build_path` and `:app` come only from `mix.exs`/`MIX_BUILD_PATH`/`MIX_BUILD_ROOT`. All operator-controlled code; a value that could redirect the delete is already RCE by design. No dependency/archive/network data feeds `config` here (`load_project` runs on the top project config, not a dep's).
