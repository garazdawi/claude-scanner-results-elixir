# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/file.ex
<!-- sha256: 04af2622b2a579860049423f7036d8b41219cfd8f0d92e6f8dd6368742da2c34 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:44:32Z_

I've now empirically verified every security-relevant control in this file. All hold up. Let me compile the report.

## Summary

This file (`File`) is a thin wrapper over Erlang's `:file` module. Its only non-trivial security logic lives in the recursive operations (`rm_rf/1`, `cp_r/3`) — symlink handling, path-cycle/DoS guards, and null-byte rejection. I tested each of these against the built-from-source `elixir` (1.21.0-dev / OTP 29) and they all behave correctly. The remaining "concerns" (path traversal, filesystem TOCTOU, read-all memory cost) are inherent to a filesystem API, documented, and are the *caller's* responsibility — `File` faithfully passes through whatever path the caller supplies and does not itself feed untrusted data into a sink.

---

# Security review: `lib/elixir/lib/file.ex`

## Summary
`File` is a faithful, thin wrapper over `:file`; its non-trivial security controls (symlink non-following in `rm_rf`, null-byte rejection, `cp_r` dereference cycle-detection, copy-into-subdirectory guard) all hold up under empirical testing, and no vulnerability is reachable through its documented public API.

## Security

`No findings`

### Ruled out

- **Poison null-byte path truncation in the un-guarded sinks** (`rm/1`, `write/3`, `read/2`, `cp/3`, `rename/2`, `mkdir_p/1`, `stat/2`, …). `assert_no_null_byte!/2` (file.ex:2425) is applied only in `cp_r/3` (line 1214) and `rm_rf/1` (line 1659), so I checked whether the other sinks are exploitable via `"/safe\0/../../etc/x"` truncation. **Basis: verified** — OTP 29's `:file` layer rejects embedded nulls uniformly: `File.rm/write/read/cp/rename/mkdir_p` on a null-containing path all returned `{:error, :badarg}`, and `/tmp/cscan_test/keep` survived `File.rm("/tmp/cscan_test/keep\0.junk")`. The two explicit guards are defense-in-depth / earlier-and-clearer errors for the functions that do string path-manipulation before reaching `:file`, not a security gap in the others.

- **`rm_rf/1` following a symlink out of the target tree** (documented property: "Symlinks are not followed but simply removed"). `safe_list_dir/2` (line 1717) uses `:elixir_utils.read_link_type` = `read_link_info` (lstat, confirmed at `elixir_utils.erl:108`), so a symlink is classified `:symlink` → `{:ok, :regular}` → `do_rm_regular` → unlinked, never traversed. **Basis: verified** — both a nested symlink (`target/link -> ../outside`) and a **top-level** symlink (`link_to_dir -> realdir`) were removed as links only; `outside/secret` and `realdir/precious` survived.

- **`cp_r/3 dereference_symlinks: true` infinite-recursion / DoS via symlink cycles.** The `MapSet` loop detector at lines 1291–1299 keys on the lexically-resolved target. **Basis: verified** — a self-loop (`selfloop -> src`) and a mutual directory cycle (`d1/to_d2 -> d2`, `d2/to_d1 -> d1`) both terminated promptly with `{:error, :eloop, _}` rather than hanging (both under the 20s timeout, exit 0). The recursive `do_cp_r` call is in tail position, so no stack blow-up either.

- **`cp_r/3` copying a directory into its own subtree (unbounded recursion).** Guarded by the `List.starts_with?(dest_parts, source_parts)` → `:einval` check (lines 1224–1225), and each directory level copies from a `:file.list_dir` snapshot taken before writing, so freshly-created entries are not re-walked. Best-effort/lexical (does not resolve symlinks), but cannot loop indefinitely. **Basis: verified reachable** (`File.cp_r("a","a/b")` hits the `:einval` branch).

- **Path traversal via `File.read/write/rm/…(untrusted_path)`.** `File` performs no confinement and does not claim to (moduledoc is explicit that `~` is not expanded and paths are relative/absolute as given); `Path.safe_relative/2` is the caller's confinement primitive. The untrusted datum here is the *path the caller chose to forward* — analogous to the excluded "victim runs `Code.eval_string` on attacker input" case. **Basis:** scope — `File` does not itself introduce disk/network-derived data into these sinks.

- **`cp_r/3 dereference_symlinks: true` reading files outside the tree** (a malicious symlink target being dereferenced and copied). This is the documented, opt-in purpose of the flag (equivalent to `cp -L`); it copies only content the invoking process can already read and requires the caller to both enable the flag and copy an attacker-populated tree. **Basis:** documented behavior, no privilege crossing.

- **Global-CWD race in `cd/1`, `cd!/1,2`, `cwd/0`.** Documented in the function docs as a global BEAM-wide hazard; `cd!/2` (line 2094) correctly saves `old` before `cd!` and restores it in an `after` block, with no leak on either failure path. Not driven by untrusted data beyond the caller-supplied path.

- **Unreachable second clause of `do_cp_r/6`** (line 1360, "If we reach this clause, there was an error…"). Every call site passes a list `acc`, so the `when is_list(acc)` head (line 1278) always matches and the catch-all never fires. **Basis:** reasoning over all three call sites (top-level `[]`, the `reduce_while` accumulator, the deref recursion) — all lists. It gates nothing security-relevant (errors already propagate as `{:error, _, _}`), so it is dead code / code-quality only, not a defeated defense.

- **Code execution, atom creation, command/shell execution, `binary_to_term` deserialization, network egress, crypto, template injection.** None present: no `Code.*`/`apply`, no `*_to_atom`/`Module.concat`, no `System.cmd`/`:os.cmd`/shell, no `binary_to_term`, no outbound requests, no secret comparisons. **Basis:** full-file read; the only `:os` call is `:os.type/0`, and the only `#{}` interpolations are into `ArgumentError`/deprecation strings with `inspect/1`-escaped values.

## Risks / caveats
- Reproductions ran on **Linux / OTP 29**. The Windows-only branches (`change_mode_windows/1,2` at lines 1540–1557, and the `major == :win32` arms of `safe_list_dir/2`) were reviewed but not executed; on Windows, `change_mode_windows` transiently adds owner-write (`+0o200`) before delete, leaving the file writable if the subsequent `:file.delete` fails — a minor, non-security robustness note on a file the caller already owns and is deleting.
- Null-byte rejection depends on the underlying OTP `:file` implementation returning `:badarg`; verified on OTP 29. A hypothetical older/patched runtime that truncated instead would make the missing `assert_no_null_byte!` on `rm/write/read/cp/rename` a real inconsistency — but that is out of the supported-runtime envelope.
- Filesystem TOCTOU is intrinsic to `rm_rf`/`cp_r`/`mkdir_p`/`touch` (as it is for `rm -rf`/`cp -r`); each recursion level re-`lstat`s its child, which bounds but does not eliminate the classic swap-a-dir-for-a-symlink window. This is a property of recursive filesystem tools generally, not a defect specific to this file, and requires an attacker with write access to a tree the victim is actively deleting/copying.

