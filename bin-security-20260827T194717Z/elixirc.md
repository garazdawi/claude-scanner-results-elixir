# /Users/lukas/code/claude-scanner/elixir/bin/elixirc
<!-- sha256: 448cc9ffdc2f23604eccfe370076b5afde6069a97e47dcb8d42fe22d1e169ed1 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/bin/elixirc
<!-- sha256: 448cc9ffdc2f23604eccfe370076b5afde6069a97e47dcb8d42fe22d1e169ed1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-27T20:00:06Z_

## 1. Summary

`elixirc` is a thin, correctly-quoted `sh` wrapper (14 non-blank lines of logic) whose only security-relevant defect is that its `readlink_f` helper runs `cd` on a relative path without scrubbing `CDPATH`, letting an attacker-set `CDPATH` redirect the script to `exec` an attacker-planted `elixir` binary (arbitrary code execution) when the launcher is invoked by a bare relative path. All hedges from the first pass have been resolved by execution and by reading the `elixir` neighbour; none were upgraded to findings (each failed closed under test).

## 2. Security

#### `CSCAN-elixirc-cdpath-cd-hijack`

- **Finding ID:** `CSCAN-elixirc-cdpath-cd-hijack`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `elixir/bin/elixirc:32` (function `readlink_f`), consequence realized at `elixir/bin/elixirc:43` (`exec "$SCRIPT_PATH"/elixir …`)
- **Class:** PATH and environment (attacker-influenceable `CDPATH`)
- **Impact:** A parent process that sets `CDPATH` in the environment and can write a `bin/` subdirectory under one of its entries can make `elixirc` execute an arbitrary attacker-supplied `elixir` binary. `readlink_f` runs `cd "$(dirname "$1")"` on the (relative) invocation path; when `$0` is a bare relative path like `bin/elixirc`, `dirname` yields `bin`, and dash resolves `cd bin` against `CDPATH` instead of the current directory. `SCRIPT_PATH` then points into the attacker's tree, and line 43 `exec`s the attacker's `elixir`. This vector survives PATH hardening: `CDPATH` is almost never scrubbed by wrappers or `sudo secure_path`, so a setup that sanitizes `PATH` but not `CDPATH` is still exploitable. The `> /dev/null` on line 32 (which exists only to suppress `cd`'s CDPATH echo) shows the resolution effect was not neutralized.
- **Evidence:** `verified`. Reproduced end-to-end under dash (`/bin/sh -> dash`). Real tree at `/tmp/cdtest/realtree/bin/{elixirc,elixir}`, attacker file at `/tmp/cdtest/evil/bin/elixir`:

  ```
  # from /tmp/cdtest/realtree, hostile CDPATH:
  CDPATH=/tmp/cdtest/evil /bin/sh bin/elixirc foo.ex
  => ATTACKER elixir ran; args: +elixirc foo.ex
  ```

  Boundary confirmation (same `CDPATH`, different invocation forms):
  - `"$PWD/bin/elixirc"` (absolute) → `REAL elixir ran` (safe — `cd /abs` ignores `CDPATH`)
  - `./bin/elixirc` (leading dot) → `REAL elixir ran` (safe — first component `.` skips `CDPATH`)
  - `bin/elixirc` (bare relative) → `ATTACKER elixir ran` (**vulnerable**)
  - `bin/elixirc` without hostile `CDPATH` → `REAL elixir ran` (clean baseline)

- **Suggested fix:** Scrub `CDPATH` before any `cd`. Smallest change: prefix the command — `CDPATH= cd "$(dirname "$1")" > /dev/null` — or add `unset CDPATH` once near the top of the script (line 7 area), which `readlink_f` then inherits. Using `cd -P` additionally is harmless but does not by itself disable `CDPATH`. Because `elixir` carries the byte-identical `readlink_f` (verified below), apply the same fix there.

### Ruled out

- **Line 43 `exec "$SCRIPT_PATH"/elixir +elixirc "$@"` — argument passthrough / word-splitting.** `verified` safe. Passing hostile args (`"a b.ex"`, `*`, `x";id;"`, `-o /etc`) through the real script to a boundary-printing stub preserved every argument exactly (`arg[2]=<a b.ex>`, `arg[3]=<*>`, `arg[4]=<x";id;">`, `arg[5]=<-o /etc>`), with the literal `+elixirc` correctly prepended. `"$@"` and `"$SCRIPT_PATH"` are both quoted; no splitting, globbing, or injection.
  - Axes examined: word-splitting on spaces, glob expansion, embedded shell metacharacters/quotes, leading-`-` values | Not examined: none.
- **Leading-`-` arguments reaching `elixir`'s argv (argument injection).** Not a finding: forwarding switches to `elixir`/the compiler is the documented purpose of this wrapper, and the caller's own argv is the trust root. The fixed literal `+elixirc` occupies the first switch position; user-supplied `-o`/`--foo` values are meant to be interpreted as options. I confirmed by reading `elixir` that `+elixirc` is consumed by the parser as a mode marker (`elixir:113–115`, `C=1`), i.e. it is not itself a passthrough sink.
- **`$(basename "$0")` inside the heredoc (line 11).** Not a sink: the command-substitution result is written only to stderr as usage text and is never re-evaluated as code.
- **Command injection (`eval`/`sh -c`/backticks).** None present in `elixirc`. All `$(...)` results (`readlink_f`, `SELF`, `SCRIPT_PATH`) are assigned and later used only inside double quotes; none are re-parsed as a command. (Note: `elixir` *does* use `eval` in `erl_set`, but that is out of scope for this file and its inputs are the wrapper's own literals plus argv.)
- **Exotic `$0` (argv0 beginning with `-`).** `verified` safe — fails closed. Executed test: launching a shebang script with `perl -e 'exec {"./rl.sh"} "-o"'` and `"-rl.sh"` resolved `SELF` correctly every time, because the kernel sets `$0` to the interpreter's script-path argument, not the caller's `argv[0]`. And in the hypothetical where `$0` literally began with `-`, `dirname "$1"`/`basename "$1"` reject it (`dirname: invalid option -- 'o'`, exit 1); under `set -e` (line 7) the script aborts before reaching the `exec` on line 43. No path to code execution or argument smuggling. **Upgraded from hedged assumption in first pass** (was "I assumed normal execution"; now executed).
  - Axes examined: dash-leading argv0 via `exec -a`/`perl exec`, dirname/basename option-parsing of `-o`, `set -e` abort behavior | Not examined: none.
- **Error handling.** `set -e` is present (line 7); `cd` failures abort rather than continue. No pipelines exist whose masked exit status could turn a failed verification into success, so absence of `pipefail` is not security-relevant here.
- **Temp files / privilege / download-and-run.** None of these constructs appear in the file.

## 3. Risks / caveats

- **Ambient `PATH` trust (residual, not scored separately).** `basename`, `dirname`, `readlink`, `cat`, `cd`, and `pwd` are all invoked by bare name and resolve through `$PATH`; `elixir` itself is correctly invoked by resolved absolute path (`"$SCRIPT_PATH"/elixir`), so it is not PATH-hijackable. The bare-name utilities are hijackable by an attacker who controls `PATH`, but that is the universal ambient-PATH exposure shared by essentially every shell script, and under this launcher's threat model the invoking user owns their own `PATH`. Not scored because it does not distinctly cross a trust boundary the way the `CDPATH` case does — but note the two share a precondition: an attacker who can set the victim's environment can reach either. The `CDPATH` finding is the more interesting one precisely because `CDPATH`, unlike `PATH`, is rarely sanitized.
- **Cross-file parity (now verified, not assumed).** I read the `elixir` neighbour: its `readlink_f` (lines 69–77) is byte-identical to `elixirc`'s (lines 31–39), including the unscrubbed `cd "$(dirname "$1")" > /dev/null`. So the `CDPATH` hijack is a shared defect, and `elixir` is the ultimate `exec` target of this wrapper (line 43). The fix should be applied consistently across `elixir`, `iex`, and `mix` to avoid a fixed-here-but-not-there gap. `iex`/`mix` were not read in this pass (scoped to `elixirc`), so parity there is stated, not verified.
- **Shell coverage.** The reproduction and the exotic-`$0` tests were performed against dash (this host's `/bin/sh`); the `CDPATH` resolution rule and `set -e`/`dirname` behavior are POSIX-specified, so bash/ksh `/bin/sh` behave equivalently, but I did not separately execute under those shells.

## 4. Did I verify this?

- **CDPATH hijack (the finding):** Yes — reproduced end-to-end under dash with a real/attacker tree; all four invocation-form boundaries confirmed. `verified`.
- **Argument passthrough safety:** Yes — pushed space/glob/metachar/dash-leading arguments through the real script to a boundary stub; all preserved exactly. `verified`.
- **Exotic `$0`:** Yes — executed `exec`-with-controlled-argv0 cases and characterized `dirname`/`basename` on leading-dash input; fails closed under `set -e`. `verified`.
- **`+elixirc` is a mode marker, not a passthrough sink:** Yes — read `elixir:113–115`.
- **Cross-file `readlink_f` parity with `elixir`:** Yes — read `elixir:69–77`, byte-identical.
- **`iex`/`mix` parity:** No — not read this pass; stated as expectation only.
- **Non-dash shells:** No — behavior is POSIX-specified but not separately executed under bash/ksh.

### Dropped in re-review

None. Every finding and ruled-out entry from the first pass is preserved; the two first-pass hedges ("I assumed" on exotic `$0`; "I reviewed only elixirc as scoped" on cross-file parity) were resolved by execution and by reading `elixir`, respectively, and both remain non-findings (fail-closed / shared-defect-now-verified).
