# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/will_recompile.ex
<!-- sha256: 1b265ba90baa9e74b03fdd3a41184966f0e292a26c9b36b0387324b31a5d2bf1 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/will_recompile.ex
<!-- sha256: 1b265ba90baa9e74b03fdd3a41184966f0e292a26c9b36b0387324b31a5d2bf1 -->

_Re-analysed on 2026-08-27_

`Mix.Tasks.WillRecompile.run/1` (41 lines) computes a set of `.mix/` directories and touches a lock file inside each. Its only sinks are `File.mkdir_p!` (line 35) and `File.touch!` (line 36). This pass re-reads the two neighbour modules the first pass leaned on — `project_stack.ex` (`reset_config_mtime/0`) and `project.ex` (`apps_paths/1`) — so the data-flow claims below are backed by the actual source rather than assumed.

Data-flow, now confirmed against the neighbour source:

- **`filename`** (line 31) `= Mix.ProjectStack.reset_config_mtime()`. Read `project_stack.ex:126-134`: the function mutates `config_mtime` in the project stack and then returns the module attribute `@manifest`, defined at `project_stack.ex:14` as the string literal `"compile.lock"`. The return value is a compile-time constant with no input dependency. Verified, not assumed.
- **`build`** (line 19) `= Mix.Project.build_path(config)` and **`app_path`** (line 28) are derived from `config = Mix.Project.config()` — the operator's own `mix.exs`.
- **umbrella `app` atoms** (line 22) come from `Mix.Project.apps_paths(config)`. Read `project.ex:468-497`: the map keys originate from one of exactly two sources — `config[:apps]` (a list the operator writes in their umbrella `mix.exs`, `project.ex:488-490`), or, when that is `nil`, `File.ls(apps_path)` over the umbrella's own `apps/` directory (`project.ex:481-486`). Both are operator-local; neither is fed by a dependency, archive, git ref, or network peer.

## Summary
`run/1` writes a constant-named lock file (`compile.lock`) into `.mix/` directories rooted at operator-controlled build/app paths. Every path component is either a verified compile-time constant (`filename`) or operator-controlled local layout (`build`, `app_path`, umbrella `app` names). No dependency-, archive-, git-, or network-derived value reaches `File.mkdir_p!` or `File.touch!`. No findings.

## Security

No findings

### Ruled out

- **Path traversal via umbrella `app` in `Path.join([build, "lib", Atom.to_string(app)])` (line 23).** Confidence: high. The `app` atoms are the keys of `Mix.Project.apps_paths/1`. Reading `project.ex:481-497`, those keys come from `File.ls(apps_path)` (default) or `config[:apps]` (operator's `mix.exs`). `File.ls` yields plain directory-entry names — an entry name cannot contain `/` on any supported filesystem and `File.ls` excludes `.`/`..`, so `Atom.to_string(app)` cannot introduce a separator or `..` segment; the alternative source is a literal atom list the operator wrote. Both are inside the operator's own trust boundary; a dependency cannot place a directory under the umbrella's `apps/` (deps live in `deps/`). No external trust boundary is crossed. *Axes examined: source of the atom (both `File.ls` and `config[:apps]` branches read in `project.ex`), separator/`..` content of a filesystem directory entry. Not examined: none.*

- **Attacker-controlled `filename` in `File.touch!` (line 36).** Confidence: high. `reset_config_mtime/0` returns `@manifest = "compile.lock"`, a string literal (`project_stack.ex:14,126-134`, read this pass). The value is constant regardless of any input. *Axes examined: the function body and the attribute definition it returns. Not examined: none.*

- **Atom creation (CWE-770) in this file.** Confidence: high. This file calls only `Atom.to_string/1` (lines 23) on atoms that already exist as `apps_paths` keys; it creates no atoms. The atom-*creation* site is `String.to_unsafe_atom/1` at `project.ex:483`, which is outside this file. Whether that site is a table-growth concern is a property of `project.ex`, not reachable through `will_recompile.ex` (which only stringifies already-interned atoms). *Axes examined: every atom-valued expression in this file. Not examined: none for this file's own behaviour.*

- **TOCTOU / concurrency on the touched lock file.** Confidence: high. `deps.compile` may invoke `will_recompile` from multiple partitions concurrently (`deps.compile.ex:112`), racing on the same `.mix/compile.lock`. `File.mkdir_p!` is idempotent on a pre-existing directory and `File.touch!` tolerates concurrent creation; all racers target the same operator-owned build path, so no privilege boundary is crossed and the worst outcome is a redundant mtime update. *Axes examined: idempotency of both sinks under concurrent creation, ownership of the target path. Not examined: none.*

- **Config resolving to a dependency's config as an entry point.** Confidence: high. Even in the hypothetical where `Mix.Project.config()` returned a dependency's config, that dependency's `mix.exs` has already executed arbitrary code during compilation (RCE-by-design for `deps.compile`, per the folder threat model's "operator is out of scope" boundary), and the sole additional effect would be relocating build artifacts within the build tree — no new capability. *Axes examined: capability delta of a dependency-scoped config. Not examined: none.*

### Did I verify this
- I read the two neighbour modules the first pass relied on: `project_stack.ex` (`reset_config_mtime/0` → constant `@manifest`, lines 126-134 and 14) and `project.ex` (`apps_paths/1` and its `umbrella_apps`/`to_apps_paths` helpers, lines 468-497). The constant-`filename` and operator-controlled-`app` claims are now source-verified, not assumed.
- I did not run a reproduction: there is no candidate sink with untrusted input to drive. The conclusion rests on static data-flow tracing whose every hop is a value confirmed to be either a compile-time constant or operator-local layout. No axis of any ruled-out entry remains untried at this file's boundary, so no ruled-out entry is upgraded to a low-confidence finding.

### Dropped in re-review
None. Every entry from the first pass is preserved; the two previously "out of scope, other file" caveats (`String.to_unsafe_atom` at `project.ex:483`; `~s`/`inspect` lock-line writes in `dep/lock.ex:52-57`) remain genuinely out of this file's reach — I confirmed `will_recompile.ex` only stringifies already-interned atoms and never writes lock-line text — and are noted here as pointers for the reviews of those files rather than as hedged assumptions about this one.
