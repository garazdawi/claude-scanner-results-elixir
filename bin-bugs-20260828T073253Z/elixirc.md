# /Users/lukas/code/claude-scanner/elixir/bin/elixirc
<!-- sha256: 448cc9ffdc2f23604eccfe370076b5afde6069a97e47dcb8d42fe22d1e169ed1 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/bin/elixirc
<!-- sha256: 448cc9ffdc2f23604eccfe370076b5afde6069a97e47dcb8d42fe22d1e169ed1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

The `elixirc` launcher is a thin, correctly-quoted delegation stub that forwards its arguments to `elixir +elixirc "$@"`. Its own body (lines 31–43) contains no argument-marshalling bug reachable through documented invocation. In this re-review I read the two callgraph neighbours the first pass hedged about — the sibling `elixir` parser and the platform-twin `elixirc.bat` — and resolved both hedges: the delegation contract is **verified correct**, and the Windows twin exhibits one **low-confidence parity divergence** (help-detection scope) now recorded as a finding rather than an unexamined assumption.

## Bugfixes

No findings in `elixirc` itself.

The operative body is lines 31–43. The two places user/environment data flows through — `readlink_f "$0"` and `exec "$SCRIPT_PATH"/elixir +elixirc "$@"` — quote every expansion (`"$1"`, `"$filename"`, `"$SCRIPT_PATH"`, `"$@"`). The `$# -eq 0` guard is a numeric test on a shell-controlled value; the `--help`/`-h` comparisons quote `"$1"`; nothing else is parsed here. Everything after `+elixirc` is forwarded verbatim.

**Delegation contract — verified (was a hedged assumption in the first pass).** The first pass asserted, without reading the neighbour, that `+elixirc` is "consumed `C=1` in `elixir`" and that compiler-switch arity is "out of scope." I read `elixir`:

- `+elixirc)` is an explicit `case` arm at `elixir:113–115` with `C=1`, so the marker `elixirc` prepends is recognized and copied (not shifted/dropped) into the `-extra` tail. **Verified, not assumed.**
- Everything after it (`-o`, `--profile time`, `--warnings-as-errors`, `.ex` files, and any trailing `--help`) falls through `elixir`'s `*)` catch-all (`elixir:190–197`), which copies the remaining argv verbatim after `-extra` and `break`s. I executed the delegation under `ELIXIR_CLI_DRY_RUN=1` with a trailing `--help`: it emerges as `… -extra +elixirc … --help`, i.e. forwarded as a program arg — it does **not** re-trigger `elixir`'s standalone-help guard (`elixir:11`, which requires `$# -eq 1`). So `elixirc` correctly does no compiler-switch parsing of its own and delegates cleanly.

## Risks / caveats

- **All compiler-switch arity lives in `elixir`, confirmed.** `elixirc` legitimately parses nothing beyond the no-arg/`--help`/`-h` case. Any arity or quoting defect in `-o`, `--profile time`, etc. would be a finding against `elixir:112–209`, not this file. This is now a verified structural fact (the catch-all forwarding path at `elixir:190–197`), not an out-of-scope hand-wave.

## Findings from resolving the Windows-parity hedge

### WINDOWS_PARITY — help-detection scope differs between `elixirc` and `elixirc.bat`  — **Upgraded from hedged assumption in first pass**
**Confidence: Low.**

The first pass declined to judge `elixirc.bat` ("cannot be judged from this file alone"). I read it. The two scripts detect the help request over **different argument scopes**:

- `elixirc` (sh, line 9): triggers usage only when `$# -eq 0` **or `$1`** is `--help`/`-h`. Help is **first-position-only**.
- `elixirc.bat` (lines 8–16): loops over **every** argument (`for %%A in (%*)`) and jumps to `:documentation` if **any** arg equals `--help`, `-h`, `/h`, or `/?`.

Consequence: the identical command line `elixirc foo.ex --help` behaves differently by platform. On Unix it runs `elixir +elixirc foo.ex --help` (the `--help` is forwarded as a program arg, per the trace above, and compilation proceeds); on Windows it prints the usage banner and compiles nothing. `elixirc.bat` additionally honours the cmd-native `/h` and `/?` help forms that the sh script does not (defensible as a platform convention, but still a documented-surface divergence).

Severity is low: no argument is silently dropped from the *compile* path, no quoting/injection issue, and both variants correctly prepend `+elixirc` and forward the original argv (`%*` / `"$@"`). It is a behavioural (UX) divergence in the documented `-h,--help` switch, not a correctness sink in argument marshalling. Marked low-confidence because I cannot execute cmd.exe in this sandbox to confirm runtime behaviour; the finding rests on static reading of both files.

## Ruled out

- **`readlink_f` has no symlink-cycle guard (recursion until the shell aborts).** Verified by execution: a `loopa → loopb → loopa` symlink cycle drives the recursion to the shell's ~1000-frame limit and aborts. Ruled out because it is not reachable through documented invocation — it requires the *installed* `elixirc` file itself to sit in a symlink loop (a broken installation), not any untrusted argv, environment value, or filename a caller supplies; `$0`/`"$@"` cannot introduce the cycle. The identical `readlink_f` ships in `elixir` (`elixir:69–77`), so this is a shared launcher property, not a divergence in this file.
  - Axes examined: reachability via argv/env/filename; symlink-loop depth behaviour; cross-file identity with `elixir`. | Not examined: none.

- **`cd "$(dirname "$1")" > /dev/null` failure swallowed.** Ruled out: `set -e` (line 7) is in force and `cd` is a simple command, so a failed `cd` aborts with non-zero status rather than continuing against a wrong working directory; the following `basename`/`pwd -P` cannot run in the wrong directory.
  - Axes examined: `set -e` armed at failure point; `cd` as simple (non-compound, non-`if`-guarded) command. | Not examined: none.

## Did I verify this?

- **`+elixirc` recognized/consumed correctly in `elixir`** — Yes. Read `elixir:113–115` (`C=1` arm) and `elixir:190–197` (catch-all forwarding); executed the delegation with `ELIXIR_CLI_DRY_RUN=1` and confirmed `… -extra +elixirc …`.
- **Trailing `--help` forwarded, not re-parsed** — Yes, executed; emerges after `-extra` and does not meet `elixir`'s `$# -eq 1` standalone-help guard.
- **`elixirc.bat` parity** — Read the file. One divergence found (help-detection scope), recorded as a low-confidence finding. Could **not** execute cmd.exe in this sandbox, hence low confidence.
- **Quoting of user values through delegation** — Yes (carried from first pass): every expansion in the body is double-quoted.
- **Ruled-out entries** — Both re-confirmed by execution (symlink-cycle depth) or by static `set -e` reasoning at the failure point; `Not examined: none` on both.

## Dropped in re-review

None. All prior findings and ruled-out entries are preserved. The two first-pass hedges were resolved rather than dropped: the delegation-parsing hedge is upgraded to a verified correctness statement (no finding), and the Windows-parity hedge is upgraded to a low-confidence `WINDOWS_PARITY` finding.
