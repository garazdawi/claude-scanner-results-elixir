# Cluster cross-file findings

#### `CSCAN-cluster-logger-report-validation-split`
- **Finding ID** — `CSCAN-cluster-logger-report-validation-split`
- **Severity** — Medium
- **Confidence** — high
- **Files involved** — `elixir` (sh): lines 133–144; `elixir.bat`: lines 126–127, sink at 143/145
- **Finding shape** — Trust-boundary / validation split (input allow-listed in file A, same sink reached in file B without the check)
- **Evidence** —
  - `elixir` (POSIX) deliberately restricts the `--logger-otp-reports` / `--logger-sasl-reports` value to the allow-list `{true, false}` **before** it ever reaches the command string:
    ```
    133  --logger-otp-reports)
    134      S=2
    135      if [ "$2" = 'true' ] || [ "$2" = 'false' ]; then
    136        ERL="$ERL -logger handle_otp_reports $2"
    137      fi
    ```
    (identical guard for `--logger-sasl-reports` at 139–143). If the value is anything other than `true`/`false`, `S=2` still consumes it but **nothing** is spliced into `ERL`. Because the only permitted values are metacharacter-free, the later unquoted expansion of `$ERL` at line 229 is safe.
  - `elixir.bat` drops the allow-list entirely and splices the raw value straight into the Erlang argument accumulator:
    ```
    126  if ""==!par:--logger-otp-reports=!  (set "parsErlang=!parsErlang! -logger handle_otp_reports %1" && shift && goto startloop)
    127  if ""==!par:--logger-sasl-reports=! (set "parsErlang=!parsErlang! -logger handle_sasl_reports %1" && shift && goto startloop)
    ```
    Note it uses bare `%1` (quotes preserved, no validation), and `%parsErlang%` is then re-parsed by `cmd.exe` at the shared sink `"%ERTS_BIN%erl.exe" %ELIXIR_ERL_OPTIONS% %parsErlang% %beforeExtra% -extra %*` (line 145; dry-run echo 143). So `--logger-otp-reports whatever` reaches `erl`'s `-logger handle_otp_reports whatever` unchecked (invalid Erlang logger config / boot misbehavior), and a value carrying `cmd` metacharacters rides the same `%parsErlang%` re-parse the per-file report already established as injectable.
- **What the per-file pass missed** — The `elixir.bat` reviewer treated every value-bearing flag uniformly as "a quote-breakout sink" and had no way to know these two flags were *supposed* to be constrained to `true`/`false`; that intent is only visible in the sibling `elixir` script, so the *missing allow-list* (as opposed to generic injection) is invisible to either single-file pass, and the bat report's finding #2 explicitly anchored on the `"%~1"` flags, not these `%1` ones.
- **Suggested fix** — Mirror the sh allow-list in the batch parser: gate the two logger flags on the value being exactly `true` or `false` before appending to `parsErlang` (e.g. `if !par2!=="true" ... else skip`), and consume the value via delayed expansion so no non-boolean value ever reaches the re-parsed `erl.exe` line.

#### `CSCAN-cluster-consumed-switch-argv-leak`
- **Finding ID** — `CSCAN-cluster-consumed-switch-argv-leak`
- **Severity** — Low
- **Confidence** — medium
- **Files involved** — `elixir` (sh): lines 106, 149–158, 200–216, 229; `elixir.bat`: lines 99, 121–131, 145
- **Finding shape** — Crypto/secret handling across files (a secret consumed/stripped in A is additionally exposed by B) / passthrough-model divergence
- **Evidence** —
  - In `elixir` (sh), Erlang distribution/boot switches (`--cookie`, `--name`, `--sname`, `--boot`, `--boot-var`, `--erl-config`, `--vm-args`, `--hidden`) are **fully consumed** (`S=…`, `erl_set`) and placed *before* the `-extra` boundary, and are deliberately **stripped from the program passthrough**. Tracing `--cookie SECRET foo.exs`: line 149–153 stores `-setcookie`/`SECRET` via `erl_set`; the copy/shift loop (200–209) and prepend (211–216) leave the post-`-extra` queue as just `foo.exs`; line 229 assembles `erl … -setcookie SECRET … -extra foo.exs`. The cookie value is **not** in the program's argv.
  - In `elixir.bat`, the final line re-appends the *entire original argument tail* after `-extra`: `… -extra %*` (line 145). Batch `%*` expands to all original arguments and is **not affected by `shift`** (the per-arg `shift` at line 99 and the flag `shift`s at 121–131 only reposition `%1`/`%2` for the scan loop, never `%*`). So for `elixir.bat --cookie SECRET foo.exs`, `parsErlang` gets `-setcookie "SECRET"` (line 123) **and** the program simultaneously receives `--cookie SECRET foo.exs` after `-extra`. Every switch the sh script strips is re-delivered to the elixir program's argv on Windows.
  - Net divergence: the distributed-Erlang auth cookie (and `--boot`/`--erl-config`/`--vm-args` file paths, `--name`/`--sname` node identity) that sh keeps out of the user program's argv are, on Windows only, additionally handed to the program as ordinary `System.argv/0` data — leaking via any code path that logs, echoes, or crash-dumps its own argv, and feeding `start_cli` unexpected `--`-switch tokens it never sees on Unix.
- **What the per-file pass missed** — The sh reviewer saw switches being stripped and the bat reviewer saw `%*`, but neither had the *other* file's model to compare against; the "these values must not reach program argv" invariant is only established by reading sh, so the Windows-only leak is structurally invisible to a single-file pass (and the parity ledger row left exactly this comparison "Unresolved").
- **Suggested fix** — Make the batch passthrough match the sh model: build an explicit post-`-extra` list of only the program/`-e`/`-r`/… arguments during the parse loop (as sh does) instead of dumping the unshifted `%*`, so consumed distribution/boot switches — especially `--cookie` — are not duplicated into the program's argv.
