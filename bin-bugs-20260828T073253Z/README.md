# Bugfix Scan Summary — `elixir/bin` launcher scripts

Scope: 9 launcher scripts (POSIX `sh` + Windows `.bat`/`.ps1` siblings for `elixir`, `elixirc`, `iex`, `mix`). No `VALIDATION.md` was produced, so findings are ranked by severity/confidence and by cross-file reach. All Windows/`.bat`/`.ps1` findings are `asserted` (no `cmd.exe`/`pwsh` in the review sandbox); POSIX findings were executed. Every surviving finding is **Low** severity except the one **Medium**, and none is a security/injection sink — the theme is Windows↔POSIX behavioural parity.

---

## 1. `mix` cannot start when the install path contains `[` or `]` (mix.ps1) — **Medium**
[`CSCAN-mix-getchilditem-wildcard-path`](./mix.ps1.md#cscan-mix-getchilditem-wildcard-path) · `mix.ps1:6` · confidence high\
`mix.ps1` resolves both its own path and `mix.bat` via `Get-ChildItem`, whose positional `-Path` is wildcard-expanded. If the install path or any parent dir contains the NTFS-legal literals `[`/`]` (e.g. `C:\Users\a[b]\...`), the bracketed segment is read as a character class, the match fails, `$mixBatPath` becomes `$null`, and `& $mixBatPath` on line 27 aborts — so `mix`, `mix deps.get`, `mix test`, etc. never launch. **Fix:** use `Split-Path -LiteralPath` / `Get-Item -LiteralPath` (wildcard-immune). This is the highest-impact finding: a total failure-to-start, not a cosmetic divergence.

## 2. Windows launchers exit `0` on refused/aborted (non-run) paths — **Low, systemic**
[`CSCAN-crossfile-windows-launcher-error-exit-zero`](./CROSS_FILE.md#cscan-crossfile-windows-launcher-error-exit-zero) (cross-file) + [`CSCAN-elixir-pipe-to-rejection-exit-zero`](./elixir.bat.md#cscan-elixir-pipe-to-rejection-exit-zero) (`elixir.bat:132→148`) + [`CSCAN-cluster-help-exit-status-parity`](./CLUSTER_root-1.md#cscan-cluster-help-exit-status-parity) (`elixirc.bat`)\
Every POSIX launcher ends non-boot/error paths with `exit 1`; every Windows sibling reaches `:end`→`endlocal`→EOF after a successful `echo`, exiting `0`. Concretely: `elixir.bat`'s `--pipe-to` rejection (a documented refusal — the VM never boots) exits `0` and prints to **stdout**; `iex.bat` usage and `elixirc.bat` no-args/help paths likewise exit `0` vs the siblings' `exit 1`. Any CI wrapper, `mix` task, or batch caller that branches on `%ERRORLEVEL%` treats a refused/aborted launch on Windows as success. The `--pipe-to` case is the confirmed instance; the help/usage cases generalise the same pattern. **Fix:** on the refusal/usage `.bat` paths, `exit /B 1` and write to stderr to match the `sh` siblings (or explicitly rule that WINDOWS_PARITY doesn't govern exit status).

## 3. erl-only switches leak into the program argv on Windows — **Low**
[`CSCAN-cluster-erl-switches-leak-into-extra-on-windows`](./CLUSTER_root-0.md#cscan-cluster-erl-switches-leak-into-extra-on-windows) (cluster: `elixir` vs `elixir.bat`) · confidence medium\
POSIX `elixir` consumes distribution/release/`--erl` switches (`--cookie`, `--name`, `--boot`, `--vm-args`, `--erl`) by `shift` and prepends the erl-native forms *before* `-extra`, so they never reach the program's argv. `elixir.bat` accumulates them into `parsErlang` **but then execs with `-extra %*`**, and cmd.exe `%*` always re-expands the *original* argv regardless of `shift`s — so every such switch is *also* re-passed after `-extra` into `elixir_cli`'s program argv. Identical command lines therefore yield different program-side arguments across platforms. **Fix:** on Windows, rebuild the trailing program args from the non-consumed tokens and emit that instead of `%*`.

## 4. `--pipe-to` switch-rejection guard is dead code (POSIX `elixir`) — **Low**
[`CSCAN-elixir-starts-with-guard-dead`](./elixir.md#cscan-elixir-starts-with-guard-dead) · `elixir:184-188` (helper `starts_with` at `94-99`) · confidence high, executed\
The guard tests `[ "$(starts_with …)" ]` — the *stdout* of `starts_with`, which only ever sets an exit status and prints nothing — so it is always false. When `--pipe-to` is given a missing directory arg and the next token is a switch (e.g. `-foo`), the intended clean diagnostic (`PIPEDIR cannot be a switch`, `exit 1`) never fires; the script instead runs `mkdir -p "-foo"` and the user sees the confusing `mkdir: invalid option -- 'f'` (fatal under `set -e`). **Fix:** test the exit status directly — `if starts_with "$RUN_ERL_PIPE" "-"; then` (and the `elif` for `RUN_ERL_LOG`).

## 5. `--logger-otp-reports` / `--logger-sasl-reports` value forwarded unvalidated on Windows — **Low**
[`CSCAN-cluster-logger-reports-value-validation-divergence`](./CLUSTER_root-0.md#cscan-cluster-logger-reports-value-validation-divergence) (`elixir:133-144` vs `elixir.bat:126-127`) · confidence high\
POSIX `elixir` only appends `-logger handle_otp_reports <v>` when `<v>` is the literal `true`/`false`, silently dropping anything else. `elixir.bat` appends unconditionally, so `--logger-otp-reports maybe` forwards `-logger handle_otp_reports maybe` to `erl.exe`, where the kernel logger config expects a boolean and mis-configures/faults. Reachable only with a contract-violating value. **Fix:** mirror the sh boolean guard in `.bat`.

## 6. `--help`/`-h` recognised in different argument positions for `elixirc` — **Low**
[`CSCAN-cluster-help-flag-position-parity`](./CLUSTER_root-1.md#cscan-cluster-help-flag-position-parity) (`elixirc:9` vs `elixirc.bat:9-16`) · confidence high\
POSIX `elixirc` honours help only as the **first** positional (`$1`); `elixirc.bat` scans **every** argument (`for %%A in (%*)`) and also accepts cmd-native `/h`, `/?`. So `elixirc foo.ex --help` prints usage and compiles nothing on Windows, but forwards `--help` and compiles on Unix. Purely a UX divergence — no argument is dropped from the compile path. **Fix:** pick one scope and apply to both.

## 7. Wildcard-only no-match invocation prints usage instead of compiling (elixirc.bat) — **Low**
[`CSCAN-elixirc-for-glob-argcount`](./elixirc.bat.md#cscan-elixirc-for-glob-argcount) · `elixirc.bat:9-17` · confidence medium\
The arg-counting `for %%A in (%*)` loop performs filename globbing, so a wildcard positional that matches nothing (e.g. `elixirc missing*.ex`) yields zero `FOR` iterations, `argc` stays `0`, and the script prints usage and exits without delegating — instead of letting the compiler emit its normal "no such file" diagnostic. Only wildcard-only, no-match invocations cross the threshold; ordinary and non-matching literal args count fine, and forwarded args (`%*`) are never corrupted. **Fix:** detect no-args with `if "%~1"==""` and scan for help flags with a `shift` loop, avoiding `FOR ... IN (%*)`.

---

### Files with no surviving findings
`iex`, `iex.bat`, `mix`, `mix.bat` — all reviewed as correctly-quoted delegation stubs; their earlier "out of scope" hedges were resolved by reading the delegation targets and confirmed clean.

### Unresolved coverage gaps (not defects — flagged in CROSS_FILE.md)
Not adjudicated because the implementation is out-of-repo or no interpreter was available: `+elixirc` handling inside `elixir.bat`'s parse loop; `Mix.CLI.main()` command dispatch (compiled stdlib, not in tree); the duplicated distribution-option pass-through after `-extra` on Windows; and all cmd.exe/PowerShell tokenisation axes (`%1` vs `%~1`, `%*` vs `shift`, `\\` collapse, `-LiteralPath` failure surface). A confirmation pass on a real Windows host is warranted for every `.bat`/`.ps1` finding above, all of which are `asserted` rather than executed.

*(Note: the shared `readlink_f` symlink-cycle recursion in the POSIX scripts was reproduced but correctly ruled out — reachable only via a corrupted install path, never through caller-supplied input.)*
