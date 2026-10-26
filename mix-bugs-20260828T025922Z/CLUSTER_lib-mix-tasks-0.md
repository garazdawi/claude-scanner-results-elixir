# Cluster cross-file findings

#### `CSCAN-cluster-uninstall-doc-name-mismatch`

- **Finding ID** — `CSCAN-cluster-uninstall-doc-name-mismatch`
- **Severity** — Low
- **Confidence** — high
- **Files involved**
  - `elixir/lib/mix/lib/mix/tasks/archive.install.ex` — `install/3`, lines 129–144 (esp. 130–132, 137)
  - `elixir/lib/mix/lib/mix/tasks/archive.uninstall.ex` — moduledoc example, line 13; delegation line 26
  - `elixir/lib/mix/lib/mix/tasks/archive.ex` — `run/1` listing, lines 24–39
  - (supporting sibling) `elixir/lib/mix/lib/mix/local/installer.ex` — `uninstall/2` name matching, lines 301–334
- **Finding shape** — State-machine / inherited-assumption inconsistency across the install→list→uninstall lifecycle (the name an installed artifact is stored under vs. the name a sibling task documents for removing it).
- **Evidence**
  - `archive.install.ex:131-132` derives the on-disk name from the archive's *internal root directory* (`archive_name = archive_name!(contents)`) and installs to `dir_dest = Path.join(Path.dirname(ez_path), archive_name)` — a directory named `APP-VERSION`, with **no `.ez` extension**. The `.ez` file (`ez_path`, line 130) is only used for its `dirname`; it is never written into the archives directory. Line 138 confirms the created artifact is `dir_dest`.
  - `archive.ex:28` lists installed archives via `Path.basename/1` over `archives/*`, so `mix archive` prints exactly those extension-less `APP-VERSION` directory names.
  - `archive.uninstall.ex:13` documents the removal command as `mix archive.uninstall archive.ez` and line 26 delegates to `Mix.Local.Installer.uninstall(Mix.path_for(:archives), "archive", argv, @switches)`.
  - `installer.ex:304-310,333-334` resolves the argument by `File.exists?(Path.join(root, name))` then `matching_package` (`name <> "-*"`). Passing `archive.ez` looks for `archives/archive.ez` (never created) and glob `archive.ez-*` (never matches), so the documented invocation fails; only the extension-less installed name (`APP-VERSION`, or bare app `APP` via the `-*` glob) resolves.
- **What the per-file pass missed** — A reviewer of `archive.uninstall.ex` alone sees a plausible `.ez` example and a correct delegation; only by reading `archive.install.ex` (which stores an extension-less directory) and `archive.ex` (which lists those names) together does it become clear the documented `.ez` argument can never match an installed artifact.
- **Suggested fix** — Update the `archive.uninstall.ex` moduledoc example to the name form actually created and listed, e.g. `$ mix archive.uninstall APP-VERSION` (matching what `mix archive` prints), rather than `archive.ez`.
