# /Users/lukas/code/claude-scanner/elixir/bin/iex
<!-- sha256: 5f5756abaf69e394de4e4e6d1c9f80bf4258c55509af4403a199abf5e7afcfdc -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/bin/iex
<!-- sha256: 5f5756abaf69e394de4e4e6d1c9f80bf4258c55509af4403a199abf5e7afcfdc -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-27_

## Summary
The `iex` launcher resolves its own directory with a `cd` that honours the caller's `CDPATH`, so an attacker-influenced `CDPATH` can silently redirect the resolution and make the launcher `exec` an attacker-supplied `elixir` binary — verified arbitrary code execution under two independent shells. The one finding stands; the two first-pass hedges (both about the sibling `elixir` script and about untested shells) have been discharged with executed evidence.

## Security

#### `CSCAN-iex-cdpath-hijack`
- **Finding ID:** `CSCAN-iex-cdpath-hijack`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `elixir/bin/iex:27` (`readlink_f`, the `cd "$(dirname "$1")"`), consumed at lines 36–38 (`SELF`/`SCRIPT_PATH`/`exec`)
- **Class:** PATH and environment (CDPATH → resolved exec path)
- **Impact:** When `iex` is launched via a relative path with a directory component whose first element is not `.`/`..` (e.g. the very common `bin/iex`), `readlink_f`'s `cd "$(dirname "$0")"` = `cd bin` is subject to `CDPATH`. If `CDPATH` names an attacker-writable directory, `cd bin` lands in `<cdpath>/bin`, `pwd -P` returns that attacker directory, `SCRIPT_PATH` becomes attacker-controlled, and line 38 execs `<cdpath>/bin/elixir` — full code execution as the invoking user. This defeats the author's deliberate safeguard of running `"$SCRIPT_PATH"/elixir` by absolute path instead of via `PATH`: the safeguard is only as trustworthy as `SCRIPT_PATH`, and `CDPATH` poisons `SCRIPT_PATH` itself. The `> /dev/null` on the `cd` (line 27) suppresses the directory name `cd` prints when it uses `CDPATH`, so the redirection is silent. `CDPATH` is exported/inherited in real settings (CI variables, `.envrc`/direnv, Docker `ENV`, systemd `Environment=`, wrapper scripts), which is where it crosses from a lower-privileged parent into this launcher.
- **Evidence:** `verified`. Setup: unmodified launcher copied to `/tmp/cdtest/real/bin/iex` with stub `real/bin/elixir`; attacker stub `/tmp/cdtest/evil/bin/elixir`.
  ```
  === dash, CDPATH=evil ===
  !!! EVIL elixir: /tmp/cdtest/evil/bin/elixir      # attacker code runs
  === control CDPATH empty ===
  REAL elixir: /tmp/cdtest/real/bin/elixir          # safe
  === bash ===
  !!! EVIL elixir: /tmp/cdtest/evil/bin/elixir      # attacker code runs
  ```
  Reproduced under **both `/bin/sh` (dash) and `bash`** in this pass (the first-pass "did not run bash" hedge is now discharged — the shell-implementation axis was tried, not assumed). `CDPATH` search for a bare relative operand is POSIX-specified (IEEE 1003.1, `cd` utility), consistent with both observed results.
- **Suggested fix:** Neutralise `CDPATH` for the resolution `cd`. Smallest class-removing change: `unset CDPATH` near the top (after `set -e`), or scope it per-call as `CDPATH= cd "$(dirname "$1")" > /dev/null`.
- **Cross-file note (now verified, not assumed):** The sibling `elixir` script that line 38 execs carries a **byte-identical** `readlink_f` (`elixir:69–77`; `diff` against `iex:26–34` reports IDENTICAL) and the same absolute-path `exec` at `elixir:219`/`245`. It therefore has the identical `CDPATH` exposure and needs the same fix. This is stated as verified fact, not a scope hedge — but it is a defect of `elixir`, tracked there, not double-counted against `iex`.

### Ruled out
- **`"$@"` passthrough to `elixir` (line 38):** documented, intended behavior — `iex` forwards every option to `elixir` and runs user code by design; the invoking user's own argv is the trust root, not a boundary crossing. `--` is inapplicable because `elixir` must parse those switches. Confirmed by reading the sibling: `elixir` is itself an argv parser (`elixir:108–209`) that consumes these switches; `iex` merely prepends `--no-halt --erl "-user elixir" +iex` and forwards. *Axes examined: intent, quoting, switch-injection into elixir's parser | Not examined: none.*
- **`$0` beginning with `-` reaching `dirname`/`basename` (lines 11, 27, 28):** requires an exotic invocation (`exec -a '-x'` or a `-`-named link). GNU `dirname -x` exits non-zero; under `set -e` the script aborts before any exec. Fails closed, no code execution — robustness bug, not security. *Axes examined: fail-closed behavior under set -e, GNU coreutils | Not examined: BSD/busybox dirname/basename option handling.* Entry remains `asserted` for the un-tried coreutils variants.
- **Missing `set -u` / `pipefail`:** the only expansions are `"$1"` and `"$@"`, both quoted and safe when empty; there is no pipeline whose masked exit status gates a security decision. No impact. *Axes examined: unset-var expansion, pipeline exit masking | Not examined: none.*
- **Quoting of `"$SCRIPT_PATH"` / `"$@"` in the `exec` (line 38):** both are double-quoted, so a `SCRIPT_PATH` or argument containing spaces/globs is not word-split. Checked and correct. *Axes examined: word-splitting, glob expansion | Not examined: none.*

## Risks / caveats
- **Bare-name helper utilities (`dirname`, `basename`, `readlink`, `pwd`, `cat`, `cd`; lines 11, 27–32):** these resolve through `PATH`, so a caller who fully controls `PATH` gets code execution here too. Not scored separately — it is the universal "invoking user controls `PATH`" trust-root property shared by essentially every shell script, not a defect specific to this file. The `CDPATH` finding is distinct because it silently defeats the script's own `$SCRIPT_PATH/elixir` safeguard through a non-obvious `cd` behavior.
- **BSD/macOS `sh`:** not executed here (no such shell in the sandbox). Two POSIX shells (dash, bash) agree, and `CDPATH` operand search is POSIX-mandated, so the shell axis is exercised but not exhaustively; the finding's confidence rests on the two shells actually run, not on the untested ones.

## Did I verify this
- **CDPATH hijack:** yes — reproduced end-to-end with the unmodified script under dash and bash; control run with `CDPATH=` is safe. Executed evidence above.
- **Sibling `elixir` carries the same flaw (first-pass hedge):** yes — `Read` the file and ran `diff` on the two `readlink_f` bodies; IDENTICAL. Upgraded from "assumed / out of scope" to verified cross-file fact.
- **Shell-implementation axis (first-pass hedge "did not run bash"):** yes — bash now run; reproduces. BSD/macOS still not run and remain on the caveat list.

## Dropped in re-review
None. The single finding and all four ruled-out entries from the first pass are preserved; no dismissal was upgraded past executed evidence, and the two hedged assumptions were discharged by reading/diffing the sibling `elixir` script and by running a second shell — not by weakening any finding.
