# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/escript.uninstall.ex
<!-- sha256: 0bec2bece5e0935a17fcf0933dd3c5e4eccbfdf80d9e9a5e78dae6750fb4cef7 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/escript.uninstall.ex
<!-- sha256: 0bec2bece5e0935a17fcf0933dd3c5e4eccbfdf80d9e9a5e78dae6750fb4cef7 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary
`escript.uninstall.ex` is a five-line task that delegates all work to `Mix.Local.Installer.uninstall/4` and then removes the Windows `.bat` wrapper. On re-review I read the two callgraph neighbours — `Mix.Local.Installer` (`installer.ex:300-339`) and `Mix.Tasks.Escript.Install` (`escript.install.ex:110-157`) — and executed the `File.rm` axes in the sandbox. All three first-pass hedges are now verified against source and/or executed behaviour; none survives as an unverified assumption, and none upgrades to a finding. Behaviour is correct and mirrors the install task.

## Bugfixes
No findings

## Risks / caveats
- The task's real logic lives in `Mix.Local.Installer.uninstall/4`. **Previously hedged as "out of scope"; now verified by reading `installer.ex:300-331`.** The function returns the removed path (`found`) on the success branch (`found && should_uninstall?(...)` → `File.rm_rf!(found); found`, lines 313-315) and `nil` on both the declined branch (`found ->` line 317-318) and the not-found branch (`true ->` after `Mix.shell().error`/`Mix.Task.rerun`, lines 320-323). The no-argument case raises via `Mix.raise` (line 326) before any return, so `path` in the caller is only ever a real path or `nil`. The `if path = ...` guard in `run/1` therefore only removes the `.bat` when an escript was actually removed. Return contract confirmed, not assumed.
- `File.rm/1` on the `.bat` path never raises: executed `File.rm("/tmp/...missing.bat")` → `{:error, :enoent}`, and `File.rm("<a directory>")` → `{:error, :eperm}`. Both error tuples are returned, not raised. The ignored tuple becoming `run/1`'s return value is inconsequential (Mix ignores task return values).

### Ruled out
- **Ignored `File.rm` error tuple crashes / misbehaves** — `File.rm(path <> ".bat")` on the non-Windows common case (no `.bat` present) returns an error tuple that is discarded harmlessly; no crash, no wrong behaviour.\
  Confidence: high. Axes examined: nonexistent-target (`{:error, :enoent}`, executed), target-is-directory (`{:error, :eperm}`, executed) | Not examined: none.
- **`.bat` removed when uninstall was declined or nothing found** — the `if path = ...` guard suppresses the removal because `uninstall/4` returns `nil` (not a path) on the `should_uninstall? == false` branch and on the not-found branch (verified at `installer.ex:317-323`).\
  Confidence: high. Axes examined: declined branch, not-found branch, no-arg (raises before return) — all read in source | Not examined: none.
- **`.bat` name mismatch with install** — install writes the wrapper at `dst <> ".bat"` where `dst = destination(basename) = Path.join(Mix.path_for(:escripts), basename)` (`escript.install.ex:123,142-144`); uninstall removes `found <> ".bat"` where `found` is `Path.join(Mix.path_for(:escripts), name)` for a direct-name match or the `name-*` wildcard match (`installer.ex:305-310,333-334`). In both resolutions the `.bat` sits adjacent to the escript file at the same directory, so the suffix concatenation targets the exact wrapper install created.\
  Confidence: high. Axes examined: direct-name match, versioned `matching_package` (`name-*`) match — both traced against install's write path in source | Not examined: none.

### Did I verify this?
- `Mix.Local.Installer.uninstall/4` return contract (path on success, `nil` on decline/not-found, raise on no-arg): **verified** by reading `installer.ex:300-331`. Upgraded from the first pass's "out of scope, read only to confirm" hedge — now a full source-traced verification.
- `File.rm/1` non-raising behaviour: **verified by execution** in the sandbox for both the missing-file and directory axes (`{:error, :enoent}` / `{:error, :eperm}`).
- `.bat` filename agreement between install and uninstall: **verified** by reading `escript.install.ex:110-144` and `installer.ex:305-334` and confirming both derive the wrapper path as `<escript-destination> <> ".bat"`.

### Dropped in re-review
None. All prior findings/ruled-out entries preserved; the three first-pass hedges ("out of scope for installer.ex", assumed `File.rm` behaviour, assumed `.bat` name match) are resolved to verified via neighbour reads and executed axes.
