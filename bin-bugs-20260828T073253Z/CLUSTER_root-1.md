# Cluster cross-file findings

#### `CSCAN-cluster-help-flag-position-parity`

- **Finding ID** — CSCAN-cluster-help-flag-position-parity
- **Severity** — Low
- **Confidence** — high
- **Files involved** — `/Users/lukas/code/claude-scanner/elixir/bin/elixirc:9`; `/Users/lukas/code/claude-scanner/elixir/bin/elixirc.bat:9-16`
- **Finding shape** — WINDOWS_PARITY (a documented switch recognized on one platform but not the other in the same position)
- **Evidence**
  - Unix `elixirc:9`: `if [ $# -eq 0 ] || [ "$1" = "--help" ] || [ "$1" = "-h" ]; then` — the help switch is only honoured when it is the **first** positional argument (`$1`).
  - Windows `elixirc.bat:9-15`: `for %%A in (%*) do ( if /I "%%A"=="--help" goto documentation ... )` — the loop scans **every** argument, so `--help`/`-h` (and `/h`, `/?`) trigger the usage text no matter where they appear.
  - Consequently `elixirc -o ebin --help` prints usage on Windows but falls through to compilation on Unix; `elixirc foo.ex -h` diverges the same way.
- **What the per-file pass missed** — A single-file reviewer of `elixirc` sees a self-consistent first-arg check, and a reviewer of `elixirc.bat` sees a self-consistent all-arg scan; only comparing the two siblings reveals the same documented switch is recognized in different argument positions.
- **Suggested fix** — Make the two agree: either scan all of `"$@"` for `-h`/`--help` in the Unix script, or restrict the `.bat` loop to the first token, so the documented help switch has one behaviour across platforms.

#### `CSCAN-cluster-help-exit-status-parity`

- **Finding ID** — CSCAN-cluster-help-exit-status-parity
- **Severity** — Low
- **Confidence** — high
- **Files involved** — `/Users/lukas/code/claude-scanner/elixir/bin/elixirc:9-29`; `/Users/lukas/code/claude-scanner/elixir/bin/elixirc.bat:16-19,36,41-43`
- **Finding shape** — WINDOWS_PARITY (same code path returns different exit status across the sibling scripts)
- **Evidence**
  - Unix `elixirc:28`: after emitting `USAGE` (for no-args or help), the script runs `exit 1` — the no-argument/help path terminates with a **non-zero** status.
  - Windows `elixirc.bat:16` (`if %argc%==0 goto documentation`) and `:documentation` (lines 19-36) end with `goto end`, and `:end` (lines 41-43) just runs `endlocal` and falls off the script, leaving `ERRORLEVEL` at **0** — the same no-argument/help path terminates with a **zero** status.
  - A wrapping build/CI script that invokes `elixirc` with no files to detect "nothing to do" gets exit 1 on Unix but exit 0 on Windows.
- **What the per-file pass missed** — The correct exit status is only definable relative to the sibling launcher's convention; neither `elixirc` nor `elixirc.bat` read in isolation reveals that the two platforms disagree on the exit code of the identical usage/no-args path.
- **Suggested fix** — Set a non-zero exit in the `.bat` documentation path (e.g. `exit /B 1` at `:end`, or `goto documentation` setting `ERRORLEVEL`) to match the Unix `exit 1`, or relax the Unix path to `exit 0` — pick one convention and apply it to both.
