# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/file.ex
<!-- sha256: 04af2622b2a579860049423f7036d8b41219cfd8f0d92e6f8dd6368742da2c34 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:08:13Z_

I've completed a thorough review, reading the entire file and verifying behaviors against the built Elixir (1.21.0-dev / OTP 29) in the sandbox. Here is the report.

## Summary

`File` is a well-tested standard-library module; I found no reachable correctness/crash bug on documented valid input, only two documentation-contract inaccuracies (both Low).

## Bugfixes

#### `CSCAN-file-cp_r-einval-same-as-source`
- **Finding ID** — `CSCAN-file-cp_r-einval-same-as-source`
- **Severity** — Low
- **Confidence** — high
- **Location** — `file.ex:1141` (doc for `cp_r/3`) vs. guard at `file.ex:1224` (`cp_r/3`)
- **Issue** — The `cp_r/3` docs list `:einval - destination is the same as or a subdirectory of source` as a typical error reason, but the only code path that produces `:einval` is `if source_parts != dest_parts and List.starts_with?(dest_parts, source_parts)`. Because it requires `source_parts != dest_parts`, the *same-as-source* case can never yield `:einval`; it falls through and returns `{:ok, …}`.
- **Class** — Documentation mismatch.
- **Impact** — A caller who reads the docs and copies a directory onto itself expecting the documented `{:error, :einval, dest}` to guard them instead gets `{:ok, [...]}`. It is a safe no-op (the `:exclusive` copy hits `:eexist` and `path_differs?/2` returns `false`, so nothing is overwritten and no data is lost), so the impact is limited to the docs promising an error path that does not exist.
- **Evidence** — `verified`:
```
$ elixir -e 'File.mkdir_p!("/tmp/t2/src"); File.write!("/tmp/t2/src/f.txt","d"); IO.inspect File.cp_r("/tmp/t2/src","/tmp/t2/src")'
cp_r onto self: {:ok, ["/tmp/t2/src"]}
```
No `:einval` is returned for `destination == source`, contradicting the doc line.
- **Suggested fix** — Reword the doc reason to `:einval - destination is a subdirectory of source` (drop "the same as or"), which matches the guard.

#### `CSCAN-file-read-raw-option-doc-copy-paste`
- **Finding ID** — `CSCAN-file-read-raw-option-doc-copy-paste`
- **Severity** — Low
- **Confidence** — high
- **Location** — `file.ex:407-408` (`read/2` docs) and `file.ex:433-434` (`read!/2` docs)
- **Issue** — The `:raw` option added to `read/2` and `read!/2` ("since v1.20") documents the option verbatim as *"a single atom to bypass the file server and only check for the file locally"* — text copied from `regular?/2`, `dir?/2`, and `exists?/2`. For a read, nothing is "checked"; the phrase "only check for the file locally" is semantically wrong for these two functions.
- **Class** — Documentation mismatch.
- **Impact** — A reader of `File.read/2`'s docs is told the option performs a *check* rather than reading the file with the file-server bypassed; only misleading, no runtime effect.
- **Evidence** — `asserted` (textual, directly in file): the identical string appears at `file.ex:407` and `file.ex:169`/`194`/`231`. The functional behavior is correct — `File.read("/tmp/probe_a.txt", [:raw])` returned `{:ok, "hello"}` in the sandbox — so only the wording is defective.
- **Suggested fix** — Replace the `read`/`read!` `:raw` description with e.g. *"bypass the file server and read the file directly from the local node"*.

## Risks / caveats

- I read the whole file (2465 lines, both halves) and exercised the higher-risk multi-clause paths (`stream!` deprecation dispatch, `cp`/`cp_r` deprecated-callback detection, `cp_r` symlink dereference incl. broken links and diamond/shared-target links, `cp_r` into-subdir vs onto-self, `touch` new/existing/tuple/integer, `read`/`exists?` with `:raw`) against the built compiler. No crash on valid input and no wrong return value surfaced.
- The two findings are documentation-contract issues, not logic defects. I could not turn up a functional bug reachable through the public API.

### Ruled out
- **`cp_r` dereference false `:eloop` on two symlinks to the same target** — I hypothesized the `MapSet` cycle-tracker (`file.ex:1294`) would flag a diamond (two sibling links → same file) as `:eloop`. `verified` this does *not* happen: the `dereference` set is threaded down a single ancestry chain but is *not* rethreaded through the `Enum.reduce_while` accumulator (`file.ex:1314`), so siblings each start from the parent's set. `File.cp_r(src, dst, dereference_symlinks: true)` with `a→f.txt` and `b→f.txt` returned `{:ok, [...b, ...a, ...f.txt, dst]}` — both copied. Real cycles (a link whose target is an ancestor in the same chain) are still caught.
- **`open/3` masking the function result via `:ok = close(io_device)` (`file.ex:1910`)** — a `close` error tuple (only with `:delayed_write`) would raise `MatchError` in the `after` block and hide the original result. `asserted` this is the documented, intended contract (docs explicitly warn against `:delayed_write` here and state success is expected); matches upstream, not a defect.
- **`rm_rf` swallowing child-deletion errors (`file.ex:1667-1674`)** — errors from child recursion are discarded (`{:error,_,_} -> acc`). `asserted` this is intentional best-effort behavior: any un-removed child makes the subsequent `rmdir(path)` fail with `:eexist`, which propagates, so `{:ok, …}` is never returned while files remain (only the reported reason may be the parent's `:eexist` rather than the child's cause).
- **`change_mode_windows` record-index arithmetic (`file.ex:1543`,`1553`)** — `elem(file_info, 3)` (access) and `elem(file_info, 7)` (mode) — `asserted` correct against the `#file_info{}` record layout `{file_info, size, type, access, atime, mtime, ctime, mode, …}`; on non-`win32`, `change_mode_windows/1`'s bodyless `if` returns `nil`, so `rm/1` yields `nil || e == e`.
- **`normalize_modes` `:utf8`/`:charlist` interaction (`file.ex:2436-2458`)** — `asserted` correct: `:charlist` sets `binary? = false` so no `:binary` default is appended, and `:utf8` maps to `{:encoding, :utf8}`; both orderings produce a consistent, valid `:file.open/2` option list.
- **`cp_r` prefix-collision false positive (`/a/b` vs `/a/bc`)** — `asserted` avoided: the check uses `List.starts_with?` on `Path.split` *components*, not `String.starts_with?` on the raw path, so `["/","a","bc"]` does not start with `["/","a","b"]`.

