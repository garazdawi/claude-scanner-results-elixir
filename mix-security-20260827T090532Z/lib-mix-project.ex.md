# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/project.ex
<!-- sha256: dd15fcfc0fa3a8798ae6392bc25f0ea82eed9942af0da709be0fb8a2fb49fd92 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/project.ex
<!-- sha256: dd15fcfc0fa3a8798ae6392bc25f0ea82eed9942af0da709be0fb8a2fb49fd92 -->

_Re-analysed on 2026-08-27 (hedge-verification pass over the first-pass report)_

This pass re-read the two callgraph neighbours the first pass had only assumed about — `Mix.ProjectStack` (`project_stack.ex`) and `Mix.CLI` (`cli.ex`) — and confirmed each hedged claim against their actual code. No claim needed upgrading to a finding; every hedge resolved to a verified statement. Details in "Did I verify this".

## Summary

`Mix.Project` is config/metadata/path-computation plumbing whose dangerous operations (compiling `mix.exs`, running a project's `project/0`, atom creation) all operate on operator-controlled configuration/environment or on data that only exists *after* by-design arbitrary dependency code has run, so no untrusted-data-to-sink path survives scoping. The three cross-file assumptions the first pass relied on — the `state_loader` task source, the single-consumption of `pop_post_config`, and the `@private_config`-drop + post-config-merge defense — are now verified against `cli.ex` and `project_stack.ex` rather than assumed.

## Security

No findings.

### Ruled out

- **`String.to_unsafe_atom(task || config[:default_task] || "run")` (line 283)** — atom creation (CWE-770). **Verified (was hedged):** `task` is the second element of the `{:cli, List.first(args)}` tuple that `Mix.CLI.load_mix_exs/2` writes via `Mix.ProjectStack.post_config(state_loader: {:cli, List.first(args)})` at `cli.ex:65`. `List.first(args)` is the operator-typed task name from `System.argv()` (`cli.ex:11`). `push_config` reads it through `Mix.ProjectStack.pop_post_config(:state_loader)`, which is implemented as `Keyword.pop(post_config, key)` (`project_stack.ex:49-54`) — a destructive pop, so the `state_loader` entry is consumed exactly once and this branch runs only for the root invocation. `config[:default_task]` is the operator's own `cli/0`/`project/0` value. At most one operator-controlled atom per `mix` run; not attacker-influenced, not unbounded. `to_unsafe_atom` confirmed to be the atom-creating form (vs `to_existing_atom`), but the input is operator-controlled. Confidence: high.

- **`Enum.map(apps, &String.to_unsafe_atom/1)` over `File.ls(apps_path)` (lines 482-483, `umbrella_apps/2`)** — filesystem-entry-to-atom. `apps_path` is `config[:apps_path]` (operator's `mix.exs`, default `"apps"`), entries are the developer's own umbrella children inside their source tree. Reaching it with attacker-chosen entries requires write access to the project tree, and atom-table exhaustion would need ~10⁶ directory entries; this `File.ls → to_unsafe_atom` shape is a deliberate, pervasive Mix idiom. No external (dependency/network/archive) source controls this directory. Confidence: high.

- **`Code.compile_file(file)` with `file = Path.expand("mix.exs")` (lines 1046, 1055, `load_project/2`)** — arbitrary code execution. This is exactly the documented Mix mechanism: `in_project/4` compiles each dependency's `mix.exs`, and "dependency build scripts run arbitrary code" is explicitly out of scope in the folder threat model. No integrity gate is expected here. Confidence: high.

- **`apply(module, state_loader, [])` (line 280) and `atom.project()` (line 1093, `get_project_config/1`)** — dynamic dispatch. `module` is the project module being pushed and `state_loader` is an internal atom (`:cli`/`:project`); this runs the project's own config functions by design, not an externally chosen module/function name. Confidence: high.

- **`@private_config` drop (lines 1091-1093) combined with `Keyword.merge(config, post_config)` (`project_stack.ex:262`)** — **Verified (was hedged): this is a *correct, live* defense, not a defect.** `get_project_config(atom)` calls `atom.project() |> Keyword.drop([:build_scm, :consolidation_path, :deps_app_path, :deps_build_path])`, so a dependency's own `project/0` cannot emit any of those four keys. Independently, `Mix.ProjectStack.push/3` computes `config = Keyword.merge(config, post_config)` at `project_stack.ex:262` — `post_config` is the **second** argument to `Keyword.merge`, so on any key collision the post-config value **wins**. The parent injects its `deps_config/1` values (`consolidation_path`, `deps_path`, `deps_build_path`, `lockfile`) as post-config via `load_project/2 → Mix.ProjectStack.post_config/1` before `push`, so even if the drop were bypassed the parent's values would override. Both halves of the defense are reachable and fire. Confidence: high.

- **`Path.join`/`Path.expand` sites (lines 316, 500, 585, 776, 785, 831, 836, 857, 888, 913-915, 941)** — every path component derives from operator `config`/`MIX_*` env vars or `File.cwd!()`. `deps_paths/1` (line 643) merely *returns* `opts[:dest]`; it is not used as a `File` sink in this file. No dependency/network/archive/git-ref string reaches a filesystem operation here. Confidence: high.

- **`build_structure/2` file ops — `File.mkdir_p!`, `File.rm_rf!`, `Mix.Utils.symlink_or_copy` (lines 940-957)** — `source` is `opts[:source] || File.cwd!()` (trusted caller/cwd) and `target` is `app_path(config)` (operator config). No external component. Confidence: high.

- **`traverse_deps_depth/5` recursion (lines 693-705)** — terminates at a `Map.merge` fixpoint over the finite, already-resolved dependency graph; `depth`/`parents` come from the trusted task caller per `deps_traversal_opts`. No unbounded loop or caller-controlled cost from untrusted input. Confidence: high.

## Risks / caveats

- **Dependency-supplied config reaching path sinks is real but dominated by by-design RCE.** A malicious dependency could set, e.g., `app: :"../../evil"` in its `project/0`, which `app_path/1` (line 857) would feed into `Path.join`. Verified by data flow: that value is produced by `get_project_config(atom) → atom.project()` (line 1093), i.e. the dependency's *own* `project/0`, which only runs after `Code.compile_file` has already executed that dependency's `mix.exs` (arbitrary code). It is therefore strictly weaker than the RCE already granted and is excluded by the folder threat model. Not treated as a finding for that reason. Confidence: high (data-flow-confirmed, not exploit-driven).

- **Not exercised end-to-end.** The verification here is data-flow tracing plus reading the two neighbour modules and confirming the `to_unsafe_atom` semantics with the toolchain; I did not drive a full umbrella or `mix deps.get`/`mix compile` run against a crafted dependency, because every candidate path resolved to operator-controlled input or by-design dependency-code execution before reaching a sink.

- **Out-of-file sinks (genuinely other files, correctly deferred).** The security-critical Mix behaviours (archive/zip-slip extraction, checksum verification, git-ref/URL argument handling, lockfile/CSV parsing) live in `mix/tasks/archive.install`, `mix/scm/git.ex`, `mix/dep/*`, and `mix/utils.ex`. These are separate files with their own review scope, not assumptions about `project.ex`'s behaviour, so deferring them is a scoping boundary rather than an unverified hedge.

## Did I verify this

- **`state_loader` task source and single-consumption** — VERIFIED by reading `cli.ex:11,65` (source = `List.first(System.argv())`) and `project_stack.ex:42-54` (`post_config` merges; `pop_post_config` is a destructive `Keyword.pop`). First pass had asserted this from a line reference only; now read directly.
- **`@private_config` drop + post-config-merge override** — VERIFIED by reading `project_stack.ex:252-283`; confirmed `Keyword.merge(config, post_config)` puts post-config as the winning (second) argument, and `get_project_config/1` (line 1093 of this file) drops the four private keys before merge. First pass had asserted the merge direction from a line reference; now confirmed from the actual argument order.
- **Malicious-dep `app:` value reaching `Path.join`** — VERIFIED by data-flow reading (line 1093 `atom.project()` runs the dep's own function post-compile); confirmed strictly weaker than granted RCE.
- **`String.to_unsafe_atom` is the atom-creating form** — VERIFIED with the toolchain (creates fresh atoms vs `to_existing_atom` rejecting unknowns), as in the first pass.
- **Not verified by execution:** no live umbrella / `deps.get` / `compile` run against a crafted dependency was performed (see caveat). No finding rests on an unexecuted claim — the report has no findings, and every ruled-out entry is grounded in read code or confirmed semantics.

## Dropped in re-review

None. The first pass reported no findings; no finding was dropped. Every ruled-out entry is preserved, with the three previously hedged cross-file assumptions now upgraded to **verified** (not to findings) after reading `cli.ex` and `project_stack.ex`.
