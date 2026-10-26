# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.build.ex
<!-- sha256: 430d2884f1fedb2ea18db4b4dc5f39cdfe7e0c916c732c083708b7d5d6aaeaf4 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.build.ex
<!-- sha256: 430d2884f1fedb2ea18db4b4dc5f39cdfe7e0c916c732c083708b7d5d6aaeaf4 -->

_Re-reviewed on 2026-08-27_

Re-review resolving the first-pass hedges ("out of scope for this file"). Two neighbour modules were read and one axis re-run:
- `Mix.Local.archive_name/1` (`lib/mix/local.ex:103`) **read** — confirmed `Path.basename() |> Path.rootname(".ez")`.
- `Mix.Project.build_structure/2` (`lib/mix/project.ex:937`) **read** — confirmed `priv` and `include` are symlinked from source into `_build/ENV/lib/APP`, so the default archive source's `priv` is itself a symlink. This makes the finding's "symlink-following is load-bearing" claim **verified fact**, not assumption.
- The `dir`-prefix traversal axis was **re-run** through the real call shape `Path.expand(target) |> Path.basename() |> Path.rootname(".ez")` across `..`, `/`, `../..`, `foo/..`, `../../evil.ez` — every result is `..`-free because `Path.expand` resolves `..` before `archive_name` ever sees the path.

## Summary

`Mix.Tasks.Archive.Build` packages the developer's own compiled `_build/ENV/lib/APP` tree (`.elixir` + `ebin` + `priv`) into a `.ez` zip via `:zip.create`. On the default path every input (CLI args, `mix.exs` config, the app's own build dir) is developer-controlled. The one behaviour that can cross a trust boundary is that `priv` globbing follows symlinks and embeds an outside-the-tree symlink target's **contents** into the archive. Entry names cannot be made to traverse (verified below), so this is content-capture, not zip-slip.

## Security

#### `CSCAN-archive.build-symlink-content-capture`
- **Finding ID** — `CSCAN-archive.build-symlink-content-capture`
- **Severity** — Low
- **Confidence** — low
- **Location** — `elixir/lib/mix/lib/mix/tasks/archive.build.ex:126` (`files_to_add/3`; `Path.wildcard("priv/**/*", …)` at line 130 → `File.read(f)` at line 133 → embedded at line 135)
- **Class** — File operations and path handling (symlink target capture / information disclosure into archive)
- **Impact** — `files_to_add/3` globs `priv/**/*` and, for every match, calls `File.read/1` and stuffs the returned *bytes* into the zip under a `priv/…` entry name. `Path.wildcard` follows symlinks and `File.read` reads through them, so a symlink inside `priv` that points **outside** the source tree (e.g. `priv/leak.txt -> /home/builder/.ssh/id_rsa`, or a symlinked directory) causes the target file's contents to be captured verbatim into the built `.ez`. On the normal path this crosses no boundary (the developer archives their own project, and — now verified — Mix itself symlinks the real `priv` into `_build` via `Mix.Project.build_structure/2:957`, so following symlinks is required for the tool to work at all). It becomes a disclosure primitive when the *source tree is untrusted* — `mix archive.build -i <downloaded/PR tree>`, or an automated pipeline that builds/publishes archives from untrusted project trees — where a planted escaping symlink exfiltrates the builder's/CI's local files into the resulting archive. There is no traversal on the *entry name* (the in-archive prefix collapses to the target basename), so the leak is content-capture, not zip-slip.
- **Evidence** — `verified`. Reproduced with the exact glob+read shape of `files_to_add/3`:
  ```
  $ ln -s /tmp/secret_outside.txt priv/leak.txt ; ln -s /tmp/secret_dir priv/dirlink
  wildcard results: ["priv/dirlink", "priv/dirlink/inside.txt", "priv/leak.txt", "priv/sub", "priv/sub/normal.txt"]
  read: {"priv/leak.txt", {:ok, "TOP-SECRET-KEY\n"}}          # symlink pointing outside the tree
  read: {"priv/dirlink/inside.txt", {:ok, "dir-secret\n"}}    # traversed a symlinked directory
  ```
  Load-bearing symlink claim upgraded from assumption to fact by reading `Mix.Project.build_structure/2` (`lib/mix/project.ex:956-958`): `for dir <- ~w(include priv), do: Mix.Utils.symlink_or_copy(Path.join(source, dir), Path.join(target, dir))`.
- **Suggested fix** — Confine captured files to the source tree instead of trusting the symlink. A blanket "skip all symlinks" would break normal builds because Mix symlinks the whole `priv`/`include` dir into `_build` (`Mix.Utils.symlink_or_copy/2`, confirmed above); so resolve each match and drop only escapers — e.g. compare `Path.expand(File.read_link!(f) or f)` against `source_path` and reject when the canonical target is not under the archive root, or use `Path.safe_relative/2` on the resolved target. At minimum, document that `archive.build` follows symlinks and should not be run over untrusted source trees.

### Ruled out

- **Zip-slip via archive entry names** — `dir` (the in-archive prefix) is `Mix.Local.archive_name(target_path)`. **Read** `lib/mix/local.ex:103-107`: `path |> Path.basename() |> Path.rootname(".ez")`. Crucially the argument is `target_path = Path.expand(target)` (line 119), i.e. already absolutised. **Re-ran** the full real shape `Path.expand(x) |> Path.basename() |> Path.rootname(".ez")` over `x ∈ {"..", "/", "../..", "foo/..", "../../evil.ez", "/a/b/../"}` → results `{"", "", "", "tmp", "evil", "a"}`, all `..`-free (`Path.expand` collapses every `..` before `archive_name` runs; the only edge is `dir=""` when target expands to `/`, and `Path.join("", "priv/x") == "priv/x"`, still non-traversing). The `f` values are relative glob results of `.elixir` / `ebin/*.{beam,app}` / `priv/**/*`, none of which can contain `..`. So no `..` can enter an entry name from either the prefix or the suffix. Extraction-side confinement (`archive.install`'s `validate_archive_path!`) is genuinely a different file and correctly out of scope; the only claim this file owns — that the builder cannot emit a traversing entry — is verified here.
  - Axes examined: prefix source (`archive_name`), `Path.expand` `..`/root/mixed collapse, empty-prefix `Path.join`, wildcard entry-name shape | Not examined: none.
- **`-o` / `-i` write/read outside CWD** — both are the operator's own command-line arguments; `Path.expand` on a developer-supplied output/input path writing/reading where the developer pointed it is the documented behaviour of these flags, not attacker-influenced.
  - Axes examined: input provenance (argv), documented `-o`/`-i` semantics | Not examined: none.
- **RCE via `Mix.Task.run(:compile, …)`** (line 70) — compiling a project executes its `mix.exs`/code, but that is inherent to Mix and out of scope (project config = RCE by design per the folder threat model); `archive.build` adds no new sink over `mix compile`.
  - Axes examined: sink novelty vs. baseline `mix compile` | Not examined: none.
- **`File.write(".elixir", elixir)` / `File.rm`** (lines 82–84) — fixed filename under `app_path`; content is `config[:elixir]` (developer config), written as data, never used as a path or evaluated.
  - Axes examined: filename constancy, content-vs-path use | Not examined: none.
- **Symlink-cycle DoS in `priv/**/*`** — verified terminating: a `priv/loop -> priv` cycle expands only to the OS symlink-follow depth (~40 levels, 82 bounded entries, ~4 ms), so no unbounded loop or memory blowup.
  - Axes examined: cycle depth bound, entry count, wall time | Not examined: none.
- **Atom exhaustion / eval / command injection** — no `String.to_atom`/`Module.concat`, no `Code.eval*`, no `System.shell`/`os.cmd`; `OptionParser.parse!` uses a fixed strict switch list (`@switches`); `String.to_charlist` creates charlists, not atoms.
  - Axes examined: atom sinks, eval sinks, shell sinks, option-parser strictness | Not examined: none.

## Risks / caveats
- The single finding is deliberately rated **Low / low-confidence**: on the *default* invocation (`mix archive.build` on one's own project) no trust boundary is crossed, and symlink-following is intentional/load-bearing (now confirmed: `Mix.Project.build_structure/2` symlinks `priv`/`include` into `_build`). It only becomes exploitable when a builder or CI job archives a source tree it does not control (`-i`, or automated archive publishing from untrusted PRs). Reasonable reviewers could file this as a "risk/caveat" rather than a finding.
- I did not run a full end-to-end `mix archive.build` inside a real Mix project (the sandbox `/work` is not a Mix project); instead I reproduced `files_to_add/3`'s exact glob+`File.read` behaviour in isolation and re-ran the `archive_name`/`Path.expand` prefix pipeline directly — both the security-relevant slices. The surrounding project/compile plumbing (`Mix.Project.*`, `Mix.Local.*`) was read (not just assumed) at the two call sites that matter.

## Did I verify this?
- `Mix.Local.archive_name/1` reduces the prefix to `basename |> rootname(".ez")` — **YES, read** `lib/mix/local.ex:103-107`.
- The prefix cannot carry `..` given `target_path = Path.expand(target)` — **YES, re-ran** the pipeline over `..`/`/`/`../..`/traversal inputs; all `..`-free.
- `priv`/`include` are symlinked into the default archive source, making symlink-following load-bearing — **YES, read** `lib/mix/project.ex:956-958` and `Mix.Utils.symlink_or_copy/3` at `lib/mix/utils.ex:683`.
- Symlink content-capture reproduces with the real glob+read shape — **YES, executed** (see Evidence).
- Extraction-side traversal defence — **NO**, lives in `archive.install`/`validate_archive_path!`, correctly out of this file; the only in-file obligation (no traversing entry emitted) is verified.

### Dropped in re-review
None. Every prior finding and ruled-out entry is retained; the sole change is that the previously hedged "out of scope" assumptions (`archive_name` shape, `priv` symlinking, prefix non-traversal) are now backed by reads of the named neighbour modules and by re-running the `Path.expand`→`archive_name` axis, so their `Not examined` lists are `none` and their evidence is `verified` rather than asserted.
