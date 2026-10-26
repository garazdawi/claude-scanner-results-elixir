# Cluster cross-file findings

#### `CSCAN-cluster-escript-install-ignores-path-option`

- **Finding ID** — `CSCAN-cluster-escript-install-ignores-path-option`
- **Severity** — Medium
- **Confidence** — high
- **Files involved** —
  - `/Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/escript.build.ex` — lines 150–151, 214–216
  - `/Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/escript.install.ex` — lines 133–138
  - (supporting, out-of-cluster) `.../lib/mix/local.ex:26–32` (`name_for/2`), `.../lib/mix/local/installer.ex:94,101,127–137` (`build/2` return consumed by `local_install`)
- **Finding shape** — Inherited assumption broken / return-shape contract mismatch across files (one file honors a config key the paired file drops).
- **Evidence** —
  - `escript.build.ex:150-151` computes the artifact path as `filename = escript_opts[:path] || script_name`, i.e. the project's `escript: [path: ...]` option *overrides* the default name, and `escript.build.ex:214-216` writes the escript to exactly that `filename`.
  - `escript.install.ex:133-138` `build/2` runs `escript.build` and then returns the location it claims the built artifact lives at: `Mix.Local.name_for(:escripts, Mix.Project.config())`.
  - `local.ex:26-32` shows `name_for(:escripts, project)` returns only `project[:escript][:name] || project[:app]` — it never consults `:path`.
  - `installer.ex:94/101` calls `local_install(module, build(module, install_spec, opts), opts)`, and `installer.ex:127-137` reads the returned path via `Mix.Utils.read_path(src, opts)`, raising `"Expected #{src} to be a local file path"` when the file is absent.
  - Net effect: for a project configured with `escript: [path: "some/dir/foo"]`, `mix escript.install` (no-arg `:project` spec) builds the escript at `some/dir/foo` but then tries to read/install it from the default `name_for` location, which does not exist → install aborts.
- **What the per-file pass missed** — A single-file reviewer of `escript.install.ex` sees `build/2` return a plausible path and cannot know that `escript.build.ex` writes the file somewhere else when `:path` is set; the divergence is only visible by comparing the two tasks' path computations against `name_for/2`.
- **Suggested fix** — Make `escript.install`'s `build/2` return the same path `escript.build` writes to, e.g. `escript_opts = Mix.Project.config()[:escript] || []; escript_opts[:path] || Mix.Local.name_for(:escripts, Mix.Project.config())`, mirroring `escript.build.ex:151`.
