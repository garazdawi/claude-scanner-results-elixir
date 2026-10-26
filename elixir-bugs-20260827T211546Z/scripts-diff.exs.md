# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/diff.exs
<!-- sha256: 818c4831769f25e726031e89c90a3464867ce9906c8ac4366810a7889795d85e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T00:04:32Z_

## Summary

Two genuine defects in `diff.exs`: `compare_dirs/2` crashes whenever the compared build directories contain subdirectories, and the BEAM-diff path returns an iolist where the `@spec` promises a `String.t()`.

## Bugfixes

#### `CSCAN-diff-file-dir-relative-path`
- **Finding ID** — `CSCAN-diff-file-dir-relative-path`
- **Severity** — High
- **Confidence** — high
- **Location** — `diff.exs:60`, `Diff.compare_dirs/2` (`common_files = Enum.reject(common_paths, &File.dir?/1)`)
- **Issue** — `relative_paths/1` (lines 165–170) returns paths *relative to `dir1`* (e.g. `"sub"`, `"sub/b.txt"`). The reject that is supposed to drop directory entries calls `File.dir?/1` on those relative paths, which resolve against the process's current working directory, not against `dir1`. Unless the caller happens to `cd` into `dir1`, a subdirectory entry like `"sub"` is not recognized as a directory, is kept in `common_files`, and is then handed to `compare_files/2` → `File.read!(Path.join(dir1, "sub"))`, which fails with `:eisdir`.
- **Class** — Missing clause / non-total function (crash on valid input due to wrong path base).
- **Impact** — The tool's primary entry point crashes on any pair of build directories that contain nested directories — which is essentially every real `_build`/`ebin` tree. The diff never completes; the user gets a `File.Error` stack trace instead of a comparison.
- **Evidence** — `verified`. With `d1`/`d2` each containing a `sub/` subdirectory:
```
$ ERL_COMPILER_OPTIONS=deterministic elixir .../diff.exs d1 d2
** (File.Error) could not read file "/tmp/difftest/d1/sub": illegal operation on a directory
    (elixir) lib/file.ex:451: File.read!/2
    diff.exs:84: Diff.compare_files/2
    diff.exs:67: anonymous fn/3 in Diff.compare_dirs/2
```
Also confirmed directly: `File.dir?("sub")` returns `false` when cwd is not the dir holding `sub`, so the entry survives the reject.
- **Suggested fix** — Check the real location: `common_files = Enum.reject(common_paths, &File.dir?(Path.join(dir1, &1)))`.

#### `CSCAN-diff-beam-diff-returns-iolist`
- **Finding ID** — `CSCAN-diff-beam-diff-returns-iolist`
- **Severity** — Low
- **Confidence** — high
- **Location** — `diff.exs:135–148`, `Diff.chunk_diff/4` for-comprehension branch (surfaced through `beam_diff/4` → `compare_files/2`)
- **Issue** — The `@spec` at line 82 declares `compare_files(...) :: :eq | {:diff, diff :: String.t()}`, and `compare_dirs/2`'s spec (line 45) declares `diff :: String.t()`. But the common BEAM-diff path returns the for-comprehension result `[[to_string(name1), ?\n, message], ...]` — a nested list (iolist), not a binary. So `{:diff, diff}` carries a non-`String.t()` value.
- **Class** — Contract violation (`@spec` disagrees with implementation).
- **Impact** — Functionally harmless in this script (the trailer uses string interpolation, which accepts iolists), but the documented contract is false: any consumer of `compare_files/2`/`compare_dirs/2` that applies `String.*` functions to `diff`, and Dialyzer analysis of every caller, is misled. It propagates a wrong type into static analysis.
- **Evidence** — `verified`. Comparing two real, differing BEAM files:
```
diff is_binary?: false
diff is_list?: true
```
The returned `diff` is a list, while the spec claims `String.t()`.
- **Suggested fix** — Coerce the branch to a binary, e.g. wrap the comprehension result with `IO.iodata_to_binary/1` (and the `beam_diff/4` result), or relax the `@spec` to `iodata()` to match reality.

## Risks / caveats
- `write_tmp/1` (lines 178–183) writes files into a `tmp/` directory under cwd and never removes them; each run leaks temp files. This is a throwaway diagnostic script, so I did not rank it as a bug, but repeated invocations accumulate `tmp-<n>` files.
- The tool shells out to `xxd` and `diff` (`System.cmd/2`, lines 156/161); absence of those binaries raises, but that is an environment precondition of the script, not a code defect.

### Ruled out
- **`true = name1 == name2` filter at line 136 raising `MatchError` instead of filtering** — would only fire if `beam_lib.chunks/2` returned differently-named chunks for the same requested `names` list on the two files; it always returns the requested chunks in the requested order (or the `with` fails to `nil`), so `name1 == name2` holds on every reachable iteration. Basis: `asserted` (verified indirectly — the real two-beam run at `chunk_diff` produced no `MatchError` and correct `atoms`/`attributes`/`debug_info` output).
- **`chunk_diff` returning `[]` being treated as truthy in the `||` chain (line 106–108)** — an empty list is truthy, but the comprehension only runs after `chunks1 != chunks2` succeeds, guaranteeing at least one differing chunk and thus a non-empty list; and even `[]` would be an acceptable (truthy) short-circuit value. Basis: `verified` (real beam run returned a non-empty list at the first `chunk_diff`).
- **`String.ends_with?(file1, ".beam")` misclassifying files (line 91)** — `file1` is a genuine filesystem path; the suffix test is correct for `.beam` detection. Basis: `asserted` (path is `Path.join(dir1, path)` from a real file that passed `File.read!`).

