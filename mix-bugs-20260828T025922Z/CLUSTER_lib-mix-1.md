# Cluster cross-file findings

#### `CSCAN-cluster-manifest-path-raises-inside-projectstack-genserver`

- **Finding ID** — `CSCAN-cluster-manifest-path-raises-inside-projectstack-genserver`
- **Severity** — Medium
- **Confidence** — medium
- **Files involved** —
  - `elixir/lib/mix/lib/mix/project_stack.ex:253-283` (`push/3`, specifically line 263)
  - `elixir/lib/mix/lib/mix/project.ex:826-837` (`manifest_path/1`), `:768-806` (`build_path/1` → `build_per_environment/1`)
- **Finding shape** — Inherited assumption broken (a function written to raise a *clean* user-facing error in the caller's process is invoked from inside a singleton GenServer, where the raise is fatal instead).
- **Evidence** —
  - `project_stack.ex:306-312` routes `push/3` through `update_state/1` → `GenServer.call(@name, {:update_state, fun})`; the closure is then run in the GenServer via `handle_call({:update_state, fun}, _from, state)` → `fun.(state)` (`:335-338`).
  - Inside that closure, `project_stack.ex:263` calls `Mix.Project.manifest_path(config)` — so this executes *in the `Mix.ProjectStack` process*, using the freshly-merged user `config`.
  - `project.ex:826-837` `manifest_path/1` calls `build_path(config)`, which reaches `build_per_environment/1` (`project.ex:795-806`); on a non-boolean `:build_per_environment` it does `Mix.raise("The :build_per_environment option should be a boolean, got: …")`.
  - The value is attacker/user-controlled and is **not** filtered: `get_project_config/1` only drops `@private_config = [:build_scm, :consolidation_path, :deps_app_path, :deps_build_path]` (`project.ex:1091-1093`), so a `mix.exs` returning `build_per_environment: "true"` flows straight into the config passed to `push`.
  - Because `Mix.Supervisor` runs these singletons with `max_restarts: 0` (per the folder threat model), a raise inside `handle_call` crashes the `ProjectStack` GenServer and tears down the whole Mix instance, and the user sees a GenServer crash report instead of the intended one-line `Mix.raise` message — and every later task fails because the singleton (holding the project stack) is gone.
- **What the per-file pass missed** — A `project.ex`-only reviewer sees `build_per_environment/1`'s `Mix.raise` as a normal clean abort; a `project_stack.ex`-only reviewer sees `Mix.Project.manifest_path(config)` as an opaque helper. Only reading both together reveals that this particular caller runs that raising code *inside* the max-restarts-0 singleton, changing a clean error into a fatal one.
- **Suggested fix** — Compute the manifest path outside the GenServer: have `Mix.Project.push/3` (project.ex:253-272) call `Mix.Project.manifest_path(config)` in the caller's process and pass the already-built `manifest_file` into `Mix.ProjectStack.push/4`, so any `Mix.raise` on malformed config surfaces as a normal error before the GenServer call. (This also removes the latent deadlock hazard: the call is only safe today because it passes an explicit `config` and thus avoids the re-entrant `Mix.Project.config/0` → `ProjectStack.peek` self-call.)

---

Note on things that checked out clean (not reported): the `Mix.SCM` behaviour in `scm.ex` matches every callback/arity implemented in `scm/git.ex` and `scm/path.ex`; `shell.ex → ProjectStack.printable_app_name` and `project.ex ↔ ProjectStack` return shapes (`push` `:ok | {:error, binary}`, `peek` `%{name:,config:}`, `pop_post_config` tuple, `config_mtime` integer) are all consistent; `release.ex`'s uses of `Mix.Project.deps_apps/0`, `build_path/1`, and `umbrella?/1` match their definitions; and `pubsub.ex`'s `Mix.Project.config/0` usage is sound.
