# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.install.ex
<!-- sha256: 0a3ac91238b38811abaa4c0520dad54b2d5a35454223d08ca9a48c29cb867f11 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:42:17Z_

Based on my analysis and runtime testing against the source-built Elixir, here is my report.

## Summary

`Mix.Tasks.Archive.Install` compiles cleanly and its path-traversal / root-consistency guards on untrusted `.ez` archives are sound (symlinks, absolute paths, `..`, backslash and NUL are all rejected before any filesystem mutation); the two genuine defects are both low-severity: an over-broad `..` substring check that rejects legitimately-built archives, and an uninformative `MatchError` plus a leftover partial extraction when a structurally-valid `.ez` lacks an `ebin` directory.

## Bugfixes

#### `CSCAN-archive.install-dotdot-substring-overreject`
- **Finding ID:** `CSCAN-archive.install-dotdot-substring-overreject`
- **Severity:** Low
- **Confidence:** high
- **Location:** `archive.install.ex:191`, `validate_archive_path!/2`
- **Class:** Unclassified (over-broad substring guard — segment check written as substring check)
- **Impact:** An archive produced by `mix archive.build` on a project whose `priv/` (or `ebin/`) contains any file whose *name* embeds `..` (e.g. `data..bak`, `v1..2.json`) builds successfully but cannot be installed: `mix archive.install` aborts with `"... is an unsafe path"`. The guard is meant to block the path *segment* `..` (directory traversal), but `String.contains?(path, "..")` also matches `..` appearing inside an ordinary filename, which is harmless. Legitimate round-trips are broken.
- **Evidence:** verified.
  ```
  $ elixir mal.exs   # builds a zip entry "foo-1/priv/data..bak" and calls install/3
  == priv legit dotdot filename ==
  RAISED: Installation failed: invalid archive file, "foo-1/priv/data..bak" is an unsafe path
  ```
  (Real traversal such as `foo-1/../evil` is still correctly rejected, so tightening the check does not weaken security.)
- **Suggested fix:** Check for `..` as a path segment rather than a substring, e.g. `".." in String.split(path, "/", trim: true)` (still after the existing backslash/NUL checks), so only an actual `..` component is rejected.

#### `CSCAN-archive.install-no-ebin-matcherror`
- **Finding ID:** `CSCAN-archive.install-no-ebin-matcherror`
- **Severity:** Low
- **Confidence:** high
- **Location:** `archive.install.ex:142`, `install/3` (`true = Code.append_path(ebin, cache: true)`)
- **Class:** Error-handling asymmetry (crash whose message tells the caller nothing) + resource leak (partial extraction left on disk)
- **Impact:** Installing a `.ez` that is a valid zip (relative paths, single root, no traversal) but does not contain the expected `<name>/ebin` directory — which passes `check_install_spec` since only the `.ez` extension is checked — extracts the archive, then fails with `MatchError: no match of right hand side value: false` because `Code.append_path/2` returns `false` for a non-existent directory. The message gives the user nothing to act on, and the extracted directory (`archives/<name>`) is left behind, so a bogus half-installed archive persists.
- **Evidence:** verified.
  ```
  $ elixir noebin.exs   # zip with single entry "pkg-1.0/README", no ebin
  * creating mixhome/archives/pkg-1.0
  MatchError: no match of right hand side value:

      false
  # and archives/pkg-1.0 remains on disk after the failure
  ```
- **Suggested fix:** Check the `ebin` directory exists (or match `Code.append_path` explicitly) and `Mix.raise` a clear message such as `"Installation failed: archive does not contain an ebin directory"`, removing `dir_dest` on that error path so no partial install is left behind.

## Risks / caveats

- I could not fetch the upstream Elixir source for a line-by-line diff (WebFetch is not permitted in this session), so I cannot state whether either finding is a regression versus a long-standing upstream choice; both are nonetheless genuine, reachable defects judged on their own behaviour.
- The absolute-path branch of `validate_archive_path!` (`Path.type(path) != :relative`) is effectively unreachable for archives produced/read by Erlang's `:zip`, which normalizes absolute entries to relative (observed: `Illegal absolute path: /etc/passwd, converting to etc/passwd`). It remains valid defense-in-depth; I did not hand-craft raw zip bytes to probe whether a malformed central directory could surface an absolute path to `:zip.extract` — that is the one traversal angle left unverified, but the extractor applies the same normalization at extract time, so no escape was demonstrated.
- Findings were exercised by driving the public behaviour callback `install/3` directly with in-memory zips under a temp `MIX_HOME`; I did not run the full `Mix.Local.Installer.install/3` CLI wrapper, which is out of this file's scope.

### Ruled out
- **Path traversal / root escape via crafted archive** — verified rejected: absolute paths, `..` segments, backslash, NUL, and mismatched roots all raise before any filesystem write (`archive_name!` runs before `File.mkdir_p!`); extraction cwd is `archives/<root>` and every entry must share `<root>`, so files stay contained. Tested with two-root, `..`, and absolute-path archives.
- **Double-nested extraction (`archives/<name>/<name>/ebin`)** — looks suspicious but is correct and consistent with `Mix.Local.archive_ebin/1` (`[path, archive_name(path), "ebin"]`); a real archive.build output installs and loads correctly (verified end-to-end).
- **`find_previous_versions/1` `if app` else-branch** — `List.first` of `String.split/2` is never `nil` (worst case `""`, truthy), so the `else []` is dead but harmless; no misbehaviour.
- **`elem(file_info, 2)` on zip entries** — `:zip.list_dir` always yields a `:file_info` tuple with `:regular`/`:directory` type for zip entries (verified via `list_dir` inspection); no crash on the type extraction.
- **`:zip.list_dir`/`extract` error handling** — malformed/empty archives fall through the `with` `else` to `Mix.raise("... no files found")` (verified with an empty zip); no uncaught decoder crash observed.

