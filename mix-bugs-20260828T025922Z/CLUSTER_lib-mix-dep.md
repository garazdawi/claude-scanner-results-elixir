# Cluster cross-file findings

#### `CSCAN-cluster-elixir-scm-manifest-name-duplicated`

- **Finding ID** — `CSCAN-cluster-elixir-scm-manifest-name-duplicated`
- **Severity** — Low
- **Confidence** — high
- **Files involved** —
  - `elixir/lib/mix/lib/mix/dep/elixir_scm.ex:9` (`@manifest "compile.elixir_scm"`), used at `:13` (`manifest/1`) and consumed within the cluster at `loader.ex:419` and `loader.ex:431`.
  - `elixir/lib/mix/lib/mix/dep/fetcher.ex:135` (hardcoded `"*/lib/#{app}/.mix/compile.elixir_scm"`).
- **Finding shape** — Inherited assumption broken / latent (a cross-file invariant — two files must agree on the manifest filename — held together only by a duplicated string literal, not a shared symbol).
- **Evidence** —
  - `elixir_scm.ex:9` defines the single source of truth for the manifest name: `@manifest "compile.elixir_scm"`, and exposes `manifest/1` (`:12`–`:14`) that joins it onto a path. The reader in the same cluster uses that accessor: `loader.ex:419` (`Mix.Dep.ElixirSCM.manifest(Path.join(opts[:build], ".mix"))`) and `loader.ex:431` (`Mix.Dep.ElixirSCM.read(Path.join(dep_opts[:build], ".mix"))`).
  - `fetcher.ex:128`–`:138` (`mark_as_fetched/1`) instead re-spells the filename literally: `Path.join("*/lib/#{app}/.mix/compile.elixir_scm")` and `File.rm/1`s the matches. Its whole purpose is to delete that manifest so `loader.ex:417` `missing_scm_manifest?/1` returns `true` and the dep is re-marked `:compile` (`loader.ex:403`) after a fetch/update.
- **What the per-file pass missed** — A single-file reviewer of `fetcher.ex` sees a self-consistent wildcard string and a single-file reviewer of `elixir_scm.ex` sees a well-encapsulated `@manifest` accessor; only comparing the two reveals that `fetcher.ex` bypasses `ElixirSCM.manifest/1` and hardcodes the name, so the two files silently share an unenforced invariant.
- **Suggested fix** — Have `mark_as_fetched/1` derive the leaf name from the module instead of duplicating it, e.g. build the glob from `Path.basename(Mix.Dep.ElixirSCM.manifest(""))` (or expose a small `@manifest`-returning helper on `Mix.Dep.ElixirSCM` and reuse it), so a rename of the manifest can't leave `fetcher.ex` silently clearing nothing and producing stale, un-recompiled dependency builds.
