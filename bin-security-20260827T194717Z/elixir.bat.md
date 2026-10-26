# /Users/lukas/code/claude-scanner/elixir/bin/elixir.bat
<!-- sha256: 6f4364d64b7b796a84d7533024e0b5694e6471b3b58bc00690ae116cd2bb6e07 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T20:02:18Z_

No `cmd.exe`/`wine` on this aarch64 Linux host, so I cannot execute the batch file; batch findings are reasoned from documented cmd.exe parsing semantics and marked `asserted`. The run above confirms the *shell* sibling is safe from the same class (unquoted `$ELIXIR_ERL_OPTIONS` word-splits into literal argv tokens `[x] [&] [echo] …` — the `&` never runs), which sharpens the parity divergence below.

---

# Security review — `elixir/bin/elixir.bat`

## Summary
The Windows launcher assembles the final `erl.exe` command line from environment- and argv-derived variables that are expanded with parse-time `%VAR%` substitution and then re-parsed by `cmd.exe`, so a `&`/`|`/`>` inside `ELIXIR_ERL_OPTIONS`, `--erl`, or a broken-out flag value becomes an injected Windows command — an exposure the POSIX-shell sibling does not have.

## Security

#### `CSCAN-elixir-bat-erl-options-cmd-injection`
- **Finding ID** — `CSCAN-elixir-bat-erl-options-cmd-injection`
- **Severity** — Medium
- **Confidence** — medium
- **Location** — `elixir.bat:145` (and the identical dry-run echo at `:143`), `:run` clause; source variable is the inherited env var, and `setlocal disabledelayedexpansion` at `:135` is what forces the vulnerable parse-time expansion.
- **Class** — PATH and environment / Command injection (Windows-specific `%VAR%` re-parse)
- **Impact** — Anyone who can set `ELIXIR_ERL_OPTIONS` in the environment that a (possibly more-privileged) `elixir.bat` invocation inherits gains arbitrary Windows command execution, not merely the documented ability to pass extra `erl` switches. A value like `foo & malware.exe` runs `malware.exe`. The dry-run branch (`ELIXIR_CLI_DRY_RUN`) offers no protection — it injects too, because `echo … %ELIXIR_ERL_OPTIONS% …` is itself re-parsed.
- **Evidence** — `asserted`. On line 145, `setlocal disabledelayedexpansion` (line 135) means `%ELIXIR_ERL_OPTIONS%` is substituted into the command *text* while the enclosing `( … )` block is parsed, then the resulting text is executed; an unescaped `&`/`|`/`>`/`)` in the value is therefore interpreted as a cmd operator (documented cmd.exe behavior; classic `CWE-78`). I could not run `cmd.exe` (aarch64 Linux, no wine). The contrast run above shows the shell version is *not* injectable: unquoted `$ELIXIR_ERL_OPTIONS` yields literal argv tokens `[x] [&] [echo] [INJECTED-IN-SH]` with no execution — so this is a genuine sh-vs-bat divergence, not shared behavior.
- **Suggested fix** — Reference the value through delayed expansion at the point of use (`!ELIXIR_ERL_OPTIONS!`), which substitutes *after* operator parsing and so treats `& | < >` literally; i.e. keep `enabledelayedexpansion` for the exec line instead of disabling it at line 135, or up-front reject `ELIXIR_ERL_OPTIONS` values containing cmd metacharacters.

#### `CSCAN-elixir-bat-argv-value-cmd-injection`
- **Finding ID** — `CSCAN-elixir-bat-argv-value-cmd-injection`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `elixir.bat:131` (`--erl` splice into `beforeExtra`) and the quoted-value flags `:121`–`:130` (e.g. `--cookie` → `-setcookie "%~1"`); shared sink at `:145`.
- **Class** — Argument injection / Command injection (Windows `%VAR%` re-parse)
- **Impact** — Argv-derived values reach the re-parsed command line through two paths: (a) `--erl "…"` is quote-stripped by `%~1` and spliced **unquoted** into `beforeExtra`, so `--erl "x & calc"` injects `calc`; (b) the per-flag `"%~1"` quoting (line 123 etc.) is broken by an embedded double-quote in the value, e.g. `--cookie a"&calc` produces `-setcookie "a"&calc"` in `parsErlang`, whose unbalanced quotes let the `&` execute when `%parsErlang%` is re-parsed at line 145. For a directly-invoked launcher the caller is the trust root (self-inflicted), so this rises to a real boundary crossing only when a higher-privileged wrapper forwards attacker-influenced data into an `--erl`/flag value.
- **Evidence** — `asserted`. Same parse-time-substitution-then-re-parse mechanism as the finding above; not runnable without `cmd.exe`. The unquoted `%~1` at line 131 and the missing quote-escaping of value args (only the *flag* token `par` is escaped, at line 100) are visible in the source.
- **Suggested fix** — Apply the same `"=\""` quote-escaping used for `par` (line 100) to every `%~1`/`%~2` value before it enters `parsErlang`/`beforeExtra`, and consume the accumulators via delayed expansion (`!parsErlang!`, `!beforeExtra!`) at the exec site so embedded operators stay literal.

#### `CSCAN-elixir-bat-erl-exe-cwd-search`
- **Finding ID** — `CSCAN-elixir-bat-erl-exe-cwd-search`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `elixir.bat:89`–`:90` (`ERTS_BIN` forced empty) → `:145` `"%ERTS_BIN%erl.exe"`
- **Class** — PATH and environment (bare-name executable resolution)
- **Impact** — With `ERTS_BIN` empty, line 145 executes the bare name `"erl.exe"`. `cmd.exe` resolves an unqualified program name by searching the current directory *before* `PATH`, so an attacker who can drop `erl.exe` into the directory a victim launches Elixir from (or into an earlier writable `PATH` entry) achieves code execution as the victim. Unix `elixir` resolves `erl` via `PATH` only and does not search the CWD, so this extra exposure is Windows-specific.
- **Evidence** — `asserted`. `ERTS_BIN` is cleared at lines 89–90 (any inherited value is overwritten) and never repopulated in this source file, so the expansion is literally `"erl.exe"`; cmd's CWD-first search for unqualified names is documented behavior. Note: release builds template `ERTS_BIN` to an absolute ERTS path, which removes this; the exposure applies to source/PATH-based installs where it stays empty.
- **Suggested fix** — Resolve `erl.exe` to an absolute path (e.g. from a validated `ERTS_BIN`/install root) rather than invoking the bare name, or launch via a fully-qualified path so CWD is never consulted.

### Ruled out
- **Leading-`-` switch injection into `erl`** — argv values are placed in *argument* position after a specific switch (`-setcookie "%~1"`, `-config "%~1"`, …) or after `-extra` (where they are data for the Elixir program), not in `erl`'s switch position; the only raw-switch splice is `--erl`, which is by-design and already covered above. No separate finding.
- **`SCRIPT_PATH` (`%~dp0`) injection at lines 140** — Windows path syntax cannot contain `"`, and the expansions are double-quoted; no metacharacter breakout is reachable.
- **`ERTS_BIN` environment injection** — cleared at line 89 before use, so an inherited hostile value cannot reach the command line (the residual CWD-search issue from it being *empty* is reported separately above).
- **Temp-file races, `sudo`/privilege changes, download-and-run, `chmod`** — none of these constructs appear anywhere in the file.

## Risks / caveats
- **Could not execute the file.** No `cmd.exe` or `wine` is available on this aarch64 Linux host, so every batch-behavior claim is `asserted` from documented cmd.exe parsing semantics rather than `verified`. The one thing I could run — the POSIX-shell contrast — confirms the sibling script does *not* share the injection, which is what makes the divergence a real finding rather than inherited behavior. A reviewer with a Windows box should confirm the three findings by setting `ELIXIR_CLI_DRY_RUN=1` and observing the emitted/executed command for a hostile `ELIXIR_ERL_OPTIONS` and `--erl` value.
- **`%*` after `-extra` (line 145).** This unquoted re-expansion of the raw argument tail is the same sink and is theoretically injectable for callers that invoke `elixir.bat` via `CreateProcess` with unquoted metacharacters in arguments (the `CVE-2024-24576`/"BatBadBut" class). I did not raise it as a distinct finding because exploitability there is governed by the *caller's* quoting, not by this file; it is nonetheless a residual concern and a further reason to prefer the delayed-expansion mitigation.
- **Parity assumption.** My parity argument rests on the sibling `elixir` shell script splicing `$ELIXIR_ERL_OPTIONS` unquoted at line 229 (confirmed by grep) and on POSIX word-splitting semantics (confirmed by the run). I did not audit the `.bat` against `elixir` flag-by-flag for shift-count parity, as that is a correctness rather than security concern.
- **Severity framing.** The command-injection findings are rated Medium/Low rather than High because for a directly-invoked launcher the argv/env trust root is usually the invoking user; they escalate to genuine privilege crossings only in wrapper/service/CI contexts where a lower-privileged party influences `ELIXIR_ERL_OPTIONS` or a forwarded flag value while a higher-privileged process runs the launcher.

