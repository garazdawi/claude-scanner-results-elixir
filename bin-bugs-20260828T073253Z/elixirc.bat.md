# /Users/lukas/code/claude-scanner/elixir/bin/elixirc.bat
<!-- sha256: 1a3d323f73598f1ef0476409c59c8dff899970704660ac97f30381aa80be57a5 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T07:39:56Z_

## Summary

`elixirc.bat` is a thin, mostly-correct delegator to `elixir.bat`, whose only real defect is that its argument-counting `FOR` loop performs filename globbing and comma/semicolon splitting, so a wildcard-only invocation that matches nothing prints usage instead of running the compiler.

## Bugfixes

#### `CSCAN-elixirc-for-glob-argcount`
- **Finding ID:** `CSCAN-elixirc-for-glob-argcount`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `elixir/bin/elixirc.bat:9-17` (the `for %%A in (%*)` counting loop and `if %argc%==0 goto documentation`)
- **Class:** Windows-specific (cmd `FOR` set expansion / tokenization)
- **Impact:** A Windows user who invokes `elixirc` with a wildcard positional pattern that currently matches no file in the working directory — e.g. `elixirc missing*.ex` — is shown the help/usage screen and the script exits without ever delegating to `elixir.bat`, instead of letting the compiler emit its normal "no such file" diagnostic. Confusing, but low-frequency and non-destructive.
- **Evidence:** `asserted`. This is a batch file and the sandbox is Linux (no `cmd.exe`), so I could not execute it. Reasoning from documented cmd semantics: in `FOR %%A IN (set)`, the set is tokenized on spaces, commas and semicolons, and any token containing `*` or `?` is expanded against the filesystem. `%*` for `missing*.ex` yields the single token `missing*.ex`; with no matching file the `FOR` body runs zero times, so `argc` stays `0`, `if %argc%==0 goto documentation` fires, usage is printed, and control falls through `:documentation` → `goto end` → `endlocal` without reaching `:run`/`call`. (Non-wildcard arguments are returned literally by `FOR`, so ordinary invocations like `elixirc foo.ex` count correctly; only wildcard-only, no-match invocations cross the `argc==0` threshold. Comma/semicolon-containing unquoted arguments are also miscounted but never cross that threshold because they still yield ≥1 token.) The actual arguments passed onward use raw `%*`, so no argument is corrupted — only the help-vs-run decision is wrong.
- **Suggested fix:** Don't derive the run/help decision from a globbing `FOR`. Detect "no args" with `if "%~1"==""` and scan for help flags over `%1..%9` (or a `shift` loop) without a `FOR ... IN (%*)`, e.g.:
  ```bat
  if "%~1"=="" goto documentation
  :scan
  if /I "%~1"=="--help" goto documentation
  if /I "%~1"=="-h"     goto documentation
  if    "%~1"=="/?"     goto documentation
  shift
  if not "%~1"=="" goto scan
  goto run
  ```
  This avoids filesystem globbing and delimiter splitting entirely.

## Risks / caveats

- **Not executable in this sandbox.** The target is a Windows batch file and the review host is Linux; none of the findings could be reproduced by running the fragment, so all evidence is `asserted` from documented `cmd.exe` behavior rather than `verified`. A confirmation run on a real Windows shell is warranted before acting.
- **Cross-file behavior not in scope.** `elixirc.bat` delegates via `call "%~dp0\elixir.bat" +elixirc %*`; the correctness of `+elixirc` handling, `-extra %*` re-passing, and switch arities lives in `elixir.bat` and was not adjudicated here.

### Ruled out
- **`call "%~dp0\elixir.bat"` double backslash** — `%~dp0` ends in `\`, yielding `...path\\elixir.bat`. cmd.exe collapses the duplicate separator, and the sibling launchers (`iex.bat:29`, and `mix.bat:7`) use the identical form, so this is idiomatic and harmless, not a defect.
- **Exit-code propagation** — after `call elixir.bat`, the script reaches `:end`/`endlocal`/EOF with no intervening command; `call` sets `ERRORLEVEL` to the callee's exit code and `endlocal` preserves it, so the compiler's status (including `--warnings-as-errors`) propagates to the caller. Same pattern as `mix.bat`/`iex.bat`.
- **`!` in filenames without delayed expansion** — the script uses plain `setlocal` (delayed expansion off), so `!` in a path is treated literally and reaches `%*` intact; this is correct, not a bug.
- **`set /A argc+=1` inside the parenthesized `FOR` block without delayed expansion** — `set /A` reads the live environment value of `argc` each iteration rather than the block-time snapshot, so the increment works despite delayed expansion being off; not a defect.
- **Help scanning all arguments vs. the POSIX `elixirc` scanning only `$1`, and help exiting `0` here vs. `exit 1` in the sh script** — real behavioral divergences from the sibling script, but the Windows launchers (`elixir.bat`) share the exit-0 help convention and treating a `--help` anywhere as a help request is a benign superset; neither breaks a documented compile invocation, so below the reporting bar.

