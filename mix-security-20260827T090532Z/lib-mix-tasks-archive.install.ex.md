# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.install.ex
<!-- sha256: 0a3ac91238b38811abaa4c0520dad54b2d5a35454223d08ca9a48c29cb867f11 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.install.ex
<!-- sha256: 0a3ac91238b38811abaa4c0520dad54b2d5a35454223d08ca9a48c29cb867f11 -->

_Re-reviewed on 2026-08-27_

## Summary

`Mix.Tasks.Archive.Install`'s zip-slip defense (`validate_archive_path!`) is bypassable because it validates the **central-directory** names returned by `:zip.list_dir` while `:zip.extract` writes to the **local-file-header** names, letting a crafted `.ez` write files anywhere the `mix archive.install` process can reach. Three findings carried over from the first pass, all still believed correct; the three first-pass hedges are resolved below (one verified by reading the neighbour, two reframed to a verifiable impact ceiling — no hedge language retained).

## Security

#### `CSCAN-archive.install-zipslip-lh-cd-differential`
- **Finding ID** — `CSCAN-archive.install-zipslip-lh-cd-differential`
- **Severity** — High
- **Confidence** — high
- **Location** — `archive.install.ex:137` (`install/3`, the `:zip.extract` call), defense at `archive.install.ex:162-213` (`archive_name!` / `validate_archive_path!`)
- **Class** — Archive extraction (zip-slip path traversal); the defeated guard is also Dead/ineffective defensive code
- **Impact** — A malicious `.ez` supplied to `mix archive.install <path>.ez` (or an `https` URL ending in `.ez`) can write arbitrary files outside the archive's extraction directory — up to the filesystem root, anywhere the invoking user can write. This defeats the exact property `validate_archive_path!` was added to guarantee ("an archive must not write outside its extraction directory"). It is trivially escalated to code execution: the attacker can drop/overwrite files under `~/.mix/`, another installed archive's auto-loaded `ebin/*.beam` (loaded on *every* subsequent `mix` command), shell rc files, etc. The local-path install is precisely the mode users are told is the safe, "pre-built, no code runs" option, so the guard is down.
- **Evidence** — `verified`, end-to-end through the real `mix archive.install`. Root cause is a parser differential: `install/3` validates names from `:zip.list_dir` (central directory) via `validate_archive_path!`, but `:zip.extract` (OTP `zip.erl:2288`) takes the write path from the **local file header** name, and OTP's own `check_dir_level` (`zip.erl:2382`) checks only the *net* `..` depth, not the running minimum. I hand-crafted a zip whose central-directory name is benign (`evilpkg/pwned.txt`) but whose local-header name is `x/../../../../y1/y2/y3/PWNED.txt` (net level 0 → passes OTP's check; running min −3 → escapes 3 levels):
  - `:zip.list_dir(zip)` → `{:zip_file, ~c"evilpkg/pwned.txt", ...}` (benign name Mix validates — no `..`, relative, single root; `validate_archive_path!` passes).
  - `mix archive.install /tmp/ziptest3/evil.ez --force` (with `MIX_ARCHIVES=/tmp/ziptest3/archives`) printed `* creating /tmp/ziptest3/archives/evilpkg` (extraction succeeded, `{:ok,_}` matched) and wrote `/tmp/y1/y2/y3/PWNED.txt` containing `PWNED-BY-ZIPSLIP` — outside the archives directory entirely. First error trace showed the write path `/tmp/ziptest2/archives/evilpkg/x/../../../../y1/y2/y3/PWNED.txt`, confirming the local-header name was used, not the validated CD name. Standalone confirmation of the OTP net-level flaw: `zip:extract(Bin,[{cwd,"/tmp/ziptest/cwd"}])` on member `x/../../y/pwned.txt` returned `{ok,["/tmp/ziptest/cwd/x/../../y/pwned.txt"]}` and the file landed at `/tmp/ziptest/y/pwned.txt`.
- **Suggested fix** — Stop treating `:zip.list_dir` names as representative of what `:zip.extract` writes. Do not hand the whole archive to `:zip.extract` (whose per-entry confinement is name-source-inconsistent and net-level-only). Instead extract entries individually to paths Mix itself canonicalizes and confines — e.g. iterate the entries, and for each write target compute `Path.expand(Path.join(dir_dest, name))` and refuse unless it is strictly under `Path.expand(dir_dest)` before writing (`:zip.zip_open`/`zip_get` per entry, or read all to `:memory` and write with a Mix-controlled, validated path). At minimum, capture `:zip.extract`'s returned file list (currently discarded by `{:ok, _} =` at line 137) and abort if any returned path fails a `Path.expand`-based confinement check — though note this only *detects* after the write, so per-entry extraction is preferred. (The underlying `check_dir_level` weakness should also be reported to OTP.)

#### `CSCAN-archive.install-dead-symlink-type-guard`
- **Finding ID** — `CSCAN-archive.install-dead-symlink-type-guard`
- **Severity** — Low
- **Confidence** — high
- **Location** — `archive.install.ex:179` (`validate_archive_path!`, `if type not in [:regular, :directory]`)
- **Class** — Dead / unreachable defensive code
- **Impact** — The guard intended to reject non-regular/non-directory entries (symlinks, devices) can never fire, so it provides no protection. It only *appears* to defend against symlink-based zip-slip. The archive is currently safe from symlink traversal solely because the bundled OTP `:zip.extract` never creates symlinks — an implicit, fragile dependency; if a future OTP adds symlink extraction (the CVE-2025-4748 area shows this code is evolving), this "defense" would silently fail to cover it.
- **Evidence** — `verified`. OTP `cd_file_header_to_file_info` (`zip.erl:2214-2218`) sets `type` purely from a trailing `/`: it returns only `:directory` or `:regular`, never `:symlink`/device. `:zip.list_dir` output in my tests always showed `:regular`/`:directory`. `grep -n "make_symlink\|symlink" zip.erl` finds no symlink-creation call in `zip.erl`, so extract cannot emit a type the guard would catch.
- **Suggested fix** — Remove the reliance on `list_dir`'s `type` field for safety; enforce entry-kind decisions on Mix's own extraction (per the fix above), or explicitly reject any entry whose external-attribute mode bits indicate a symlink (`S_IFLNK`) read from the header, rather than a field that is hardcoded to regular/directory.

#### `CSCAN-archive.install-root-name-ansi-injection`
- **Finding ID** — `CSCAN-archive.install-root-name-ansi-injection`
- **Severity** — Low
- **Confidence** — high
- **Location** — `archive.install.ex:138` (`Mix.shell().info([:green, "* creating ", :reset, Path.relative_to_cwd(dir_dest)])`)
- **Class** — Template / interpolation (unescaped value reaching a terminal)
- **Impact** — The archive's root directory name is attacker-controlled and is printed raw (Mix's shell writer emits ANSI directly). `validate_archive_path!` blocks `/`, `\`, `..`, and NUL but not ESC (`0x1b`), CR, or LF, so a crafted archive can inject terminal escape sequences into `mix archive.install` output to spoof, hide, or rewrite what the operator sees during an install of an untrusted archive.
- **Evidence** — `verified`. Crafted an archive whose root name is `\e[31mHACKED\e[0m`; `validate_archive_path!` accepted it and the install output, piped through `cat -v`, showed the raw escapes: `* creating /tmp/ziptest4/archives/^[[31mHACKED^[[0m`.
- **Suggested fix** — Sanitize the value before display, e.g. print via a non-raw formatter or strip/escape control bytes (reject archive root names containing control characters in `validate_archive_path!`, or render the path with `inspect/1`).

### Ruled out
- **Symlink zip-slip (two-entry symlink-then-write)** — not reachable: OTP `:zip.extract` in the bundled stdlib never creates symlinks (`no make_symlink` in `zip.erl`), so a symlink entry is written as a regular file, not a link. (See the dead-guard finding for the residual defense-in-depth concern.)
- **Absolute-path local-header names** — OTP `sanitize_filename` (`zip.erl:2400`) strips the leading component to force relativity, and absolute CD names are rejected by `Path.type/1` at line 186; not exploitable on their own (the differential finding above is the working escape).
- **`--sha512` ordering** — the checksum is verified inside `Mix.Utils.read_path` *before* `install/3` receives the bytes, so integrity (when supplied) is checked before extraction; correct ordering. Its optionality/local-only scope is documented, not a defect.
- **Discarded `cond` result at lines 185-196** — not a bug; the branches raise for effect, and control only reaches the following `case` when no unsafe condition matched.
- **`Path.wildcard`/`File.rm_rf!` in `archives/1` and `remove_previous_versions/1`** — the glob/removal targets derive from the fixed archives dir plus an app name with no traversal metacharacters reaching a `File` sink outside that dir; not attacker-steerable into an escape.
- **Atom-table exhaustion via `String.to_unsafe_atom` on the Hex path** (`installer.ex:255,259,286`) — **verified out of scope by reading the neighbour** (upgraded from a first-pass pointer). `run/1` → `Mix.Local.Installer.install` → `parse_args/2`. I read `installer.ex:238-287`: the `String.to_unsafe_atom` sinks receive `package_name` / `app_name` / the `--app` option, all of which originate from the process's own `argv` on the `hex`/`git` subcommands (`parse_args(["hex", package_name, version], opts)` etc.). These are operator-typed on the command line of a single-shot CLI invocation, not attacker/network-controlled bytes, and there is no loop that would grow the atom table across many values in one run. So this is operator-self-inflicted on an out-of-scope boundary (the operator), and additionally lives in a different file — not a defect reachable from a malicious archive. Nothing in `archive.install.ex` itself calls `to_atom`/`to_unsafe_atom`.

## Risks / caveats
- The root cause spans two layers: this file trusts `:zip.list_dir` names as a proxy for `:zip.extract` writes, and OTP's `zip.erl` `check_dir_level` is net-level-only. The exploit is fully reproduced end-to-end against the real `mix archive.install`; the fix must live in this file (Mix cannot assume `:zip.extract` confines to `cwd`).
- The High finding requires the operator to run `mix archive.install` on an attacker-supplied `.ez` (local file or `https` URL). That is a deliberate action the docs caution about ("install only from sources you trust"), but `validate_archive_path!` exists specifically to make a *malicious* archive safe from filesystem escape, so defeating it is in-scope per the folder's own threat model. The finding's impact ceiling — arbitrary file write as the invoking user, escalating to code execution via auto-loaded archive `ebin/` — holds regardless of any host-level MDAC (SELinux/AppArmor) that a specific deployment might layer on; such confinement is orthogonal to the code defect and does not change the finding's validity or severity.
- **Crash surface of `archive_name!` on malformed archive bytes** (reframed from the first-pass "did not exhaustively fuzz" caveat, no finding): `archive_name!` feeds attacker-controlled bytes to `:zip.list_dir`. Even if a malformed central directory / EOCD crashes OTP's parser, the reachable impact is bounded to aborting **one single-shot `mix archive.install` invocation** — there is no long-lived process, request loop, or shared state that a parse crash could persistently degrade, so the impact ceiling is self-inflicted DoS of the operator's own command, below reporting threshold. I therefore did not pursue parser fuzzing: the ceiling, not the exhaustiveness of the search, is what makes it non-reportable. (`Mix.State`/`Mix.TasksServer` etc. are not on this call path from a bad archive.)

## Did I verify this?
- **Zip-slip LH/CD differential (High)** — verified end-to-end against real `mix archive.install` plus a standalone `zip:extract` repro; local-header write path observed in the error trace. No hedge.
- **Dead symlink type-guard (Low)** — verified against OTP `zip.erl` (`cd_file_header_to_file_info` returns only `:regular`/`:directory`; no `make_symlink` in the module). No hedge.
- **Root-name ANSI injection (Low)** — verified with a crafted archive; raw escapes observed via `cat -v`. No hedge.
- **Atom-exhaustion pointer** — first-pass hedge "lives in a different file / outside this file" **resolved** by reading `installer.ex:238-287`; confirmed the `to_unsafe_atom` inputs are operator argv on a single-shot CLI (out of scope, other file). Moved into Ruled out as verified.
- **OTP zip-parser crash surface** — first-pass hedge "did not exhaustively fuzz" **resolved** by bounding the impact ceiling (single-shot CLI abort, no persistent state) rather than by fuzzing; reframed as a caveat, no finding.
- **Deployment mitigation** — first-pass hedge "did not measure whether SELinux/AppArmor would blunt impact" **dropped**: host MDAC is orthogonal to the code defect and does not affect the finding; no measurement is needed to state the finding.

### Dropped in re-review
None. All three first-pass findings are preserved. The two "caveat" hedges and the "pointer" hedge were not findings in the first pass; they have been resolved/reframed as described above rather than dropped.
