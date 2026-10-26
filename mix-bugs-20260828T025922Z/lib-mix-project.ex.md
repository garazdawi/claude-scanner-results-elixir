# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/project.ex
<!-- sha256: dd15fcfc0fa3a8798ae6392bc25f0ea82eed9942af0da709be0fb8a2fb49fd92 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/project.ex
<!-- sha256: dd15fcfc0fa3a8798ae6392bc25f0ea82eed9942af0da709be0fb8a2fb49fd92 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

## Summary

`Mix.Project` is the project-stack core: config accessors, umbrella app resolution, build/deps path computation, and dependency-tree traversal. On re-review I resolved every "outside this file" hedge from the first pass by (a) reading the sole callgraph neighbour the hedges pointed at — `Mix.ProjectStack` (`project_stack.ex`) — and (b) executing the two convergence/idiom claims and the one contract-boundary crash claim against a live Erlang/OTP toolchain. No hedge survived as an assumption. I found **no reachable defect through the documented public API**. Two first-pass caveats are downgraded to fully-ruled-out with executed evidence; one contract-boundary caveat is retained (verified-to-raise, but only reachable by violating the documented input contract).

## Bugfixes

No findings.

## Risks / caveats

- **Contract-boundary crash in `deps_path/1` (retained, verified-to-raise, out of documented contract).** `deps_path/1` (lines 583–586) does `Path.expand(System.get_env("MIX_DEPS_PATH") || config[:deps_path])`. Unlike `build_path/1`/`do_build_path/1`, which fall back to `"_build"` when `config[:build_path]` is nil, `deps_path/1` has no fallback. I confirmed by execution that `Path.expand(nil)` raises `FunctionClauseError` (not a graceful error). This is reachable only by calling `deps_path([])` or any config lacking `:deps_path`; every real project config carries `:deps_path` via `default_config/0` (line 1082), and `config/0` returns `default_config()` when no project is on the stack. So the crash requires passing a hand-built keyword list that is not a valid project config — a violation of the implicit input contract, not documented usage. **Confidence: low** (that this is a *defect* — it is verified to raise, but only outside the contract; classified under ERROR_PROPAGATES_NOT_BADMATCHES as a `FunctionClauseError` rather than `Mix.raise`, mitigated by contract).

- **`apps_paths/1` raises `Mix.NoProjectError` off-stack (retained, low).** `apps_paths/1` (line 470) uses `{:apps_paths, Mix.Project.get!()}` as its cache key. `get!/0` raises `Mix.NoProjectError` when no project is on the stack. Passing an umbrella-shaped config while nothing is pushed therefore raises rather than returning a value. This is an unusual call pattern (the config normally comes from the pushed project) rather than documented usage. **Confidence: low.**

## Ruled out

- **`to_apps_paths/2` filter-assignment silent-inclusion (lines 492–497).** The comprehension relies on the non-generator qualifier `path = path_with_mix_exs_otherwise_warn(app, apps_path)` acting as a truthiness filter so stray files/dirs (which return `nil`, lines 514/518) are dropped rather than inserted as `{app, nil}`. **Executed:** `for x <- [1,2,3,4], y = (if rem(x,2)==0, do: x*10, else: nil), do: {x,y}, into: %{}` → `%{2 => 20, 4 => 40}` — nil rows filtered. Correct.
  - Axes examined: qualifier-truthiness-filter semantics on `nil` value | Not examined: none.

- **`traverse_deps_depth/5` `:infinity` non-convergence (lines 689–705).** First pass left this as "rests on Mix's dep-convergence invariant enforced outside this file; did not construct a live diamond reproduction." **Upgraded to fully verified by execution.** I read the neighbour to confirm the data shape (`Mix.Dep` struct: `deps: [t]`, `opts: keyword`, `app: atom` — `dep.ex:57-81`), then ran a faithful copy of `traverse_deps_depth/5` against (1) a diamond where app `:foo` appears under two parents with **differing** `opts[:dest]` ("x" vs "y"), and (2) a **self-cycle** (`z` depends on `z`). Both terminated: diamond → `%{a: "a", b: "b", foo: "y"}`, self-cycle → `%{z: "z"}`. Termination does **not** depend on the dep-convergence invariant the first pass worried about: `all_deps` is a finite set, so the `parents` key-set grows monotonically and stabilizes in ≤ N iterations; once the key-set is fixed, `children` is a deterministic pure function of the (now-fixed) qualifying set, so `Map.merge(parents, children)` reaches the `^parents` fixpoint within one further iteration regardless of value stability. The first-pass concern that "differing `opts[:dest]` under two parents could fail to converge" is refuted by the executed diamond case. Terminates.
  - Axes examined: differing per-app `fun` value across parents (dest divergence); self-referential cycle; monotonic key-set growth to fixpoint | Not examined: none.

- **Delegations to `Mix.ProjectStack` (`project_file/0` line 361, `parent_umbrella_project_file/0` line 371, `config_files/0` lines 411–413, `config_mtime/0` lines 433–435).** First pass declined these as "outside this review's scope." **Verified by reading `project_stack.ex`:** `project_file/0` (ps:152–158) returns `h.file` (binary) or `nil`; `parent_umbrella_project_file/0` (ps:160–166) returns the second stack frame's file or `nil`; `config_files/0` (ps:144–150) returns `h.config_files` (list) or `[]`; `config_mtime/0` (ps:104–124) returns an integer (≥ `@minimum_mtime`), lazily computing from `config_files` when cached mtime is nil. All return the `@spec`-documented shapes (DOCUMENTED_RETURN_SHAPE satisfied); each has an `[] ->` clause so an empty stack does not raise. No defect in the delegating boundary or the delegate.
  - Axes examined: return-shape conformance vs `@spec`; empty-stack (`[]`) branch; nil-vs-value branches | Not examined: none.

- **`push/3` duplicate-name handling (lines 262–271).** `Mix.ProjectStack.push/3` (ps:252–283) returns `{:error, existing_file}` on a name collision (via `find_project_named/2`, ps:288–294), which `push/3` here converts to a `Mix.raise` with both file paths. Documented `:ok | {:error, file}` contract honoured; no bad-match. Correct.
  - Axes examined: collision branch (`{:error, binary}`), `:ok` branch | Not examined: none.

## Did I verify this?

- **Read** the target file in full and the one callgraph neighbour the hedges pointed at, `elixir/lib/mix/lib/mix/project_stack.ex`, plus the `Mix.Dep` struct/`cached/0` definition in `dep.ex` to confirm the shape traversed by `traverse_deps_depth/5`.
- **Executed** three reproductions on the sandbox Erlang/OTP + Elixir toolchain: (1) the comprehension filter-assignment idiom (confirmed nil-drop); (2) a faithful `traverse_deps_depth/5` copy over a differing-dest diamond and a self-cycle (both converged); (3) `Path.expand(nil)` (confirmed `FunctionClauseError`).
- Every prior "outside this file" hedge is now backed by either a read of the delegate/neighbour or an executed result; no assumption remains asserted.

## Dropped in re-review

- **Dropped the standalone "`traverse_deps_depth/5` `:infinity` convergence (analyzed, not exhaustively run)" caveat** as a *risk*. New evidence: the executed diamond (differing `opts[:dest]`) and self-cycle reproductions both terminate, and the finiteness argument (monotonic key-set to fixpoint) shows termination is independent of the dep-convergence invariant. It moves from "risk resting on an external invariant" to the **Ruled out** section as verified — a downgrade, safe for later passes to re-examine.
- **Dropped the "Delegations not audited here" caveat** as an open item. New evidence: `project_stack.ex` read confirms all four delegates return their documented shapes with empty-stack guards. Moved to **Ruled out** as verified.
