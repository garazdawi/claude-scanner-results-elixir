# /Users/lukas/code/claude-scanner/elixir/bin/iex.bat
<!-- sha256: c4c9c3601401b600043def0b209d6eb0627a1ae8449a78ade3dca310541b9557 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/bin/iex.bat
<!-- sha256: c4c9c3601401b600043def0b209d6eb0627a1ae8449a78ade3dca310541b9557 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-27_

# Security Review — `elixir/bin/iex.bat`

## Summary
This `cmd.exe` batch wrapper for `iex` evaluates the caller's first argument inside quote-imbalanced `if` comparisons (lines 8–11) and forwards all arguments verbatim via `%*` into a `call` to `elixir.bat` (line 29). Both sites re-parse `cmd` metacharacters and can turn an attacker-influenced argument into command execution. In this re-review I read the delegate `elixir.bat` in full to resolve the first-pass cross-file hedge; the passthrough concern is confirmed to continue past `iex.bat` (elixir.bat itself re-forwards `%*` to `erl.exe`), but the injection point that this file owns is the `call` on line 29, which stands on its own.

## Interpreter
This file is a **Windows `cmd.exe` batch script** (`@echo off`, `setlocal`, `goto`, `%~dp0`, `%*`). It is not `sh`/`bash`; `cmd`'s percent-expansion-then-reparse rules apply. Delayed expansion is **not** enabled in `iex.bat` (`setlocal` with no `EnableDelayedExpansion`), so `!VAR!`-based attacks are inert in this file under default `cmd` configuration. I am reviewing on Linux and **cannot execute `cmd.exe`**; the sandbox has an Erlang/OTP toolchain but no way to run a Windows batch interpreter, so the `cmd`-parsing axes below cannot be tried and every finding remains `asserted` at low/medium confidence. Reading `elixir.bat` clarified what the delegate does with the forwarded arguments but — per the re-review rules — does not by itself clear any execution axis.

## Security

#### `CSCAN-iex-if-quote-injection`
- **Finding ID:** `CSCAN-iex-if-quote-injection`
- **Severity:** High
- **Confidence:** medium
- **Location:** `iex.bat:8-11`, the `:setlocal` help-detection block (`if /I ""%1""==""--help""` and the three sibling `if` lines)
- **Class:** Command injection (Windows-specific batch argument injection)
- **Impact:** If `iex.bat` is invoked with an attacker-influenced **first argument** that reaches `%1` unquoted (see delivery note), an embedded `cmd` metacharacter (`&`, `|`, `<`, `>`) is parsed as command syntax and arbitrary commands run with the privileges of whoever launched `iex.bat`. A value meant to be *data* — a filename from a directory listing (`&` is a legal Windows filename character), a node name, or a path chosen by an upstream automation step — becomes code.
- **Evidence:** `asserted`. The template is `if /I ""%1""==""--help"" goto documentation`. There are **two** literal quotes before `%1`, so after percent-expansion `%1`'s text begins in an *unquoted* parse context. With `%1` = `&calc&`, the line expands to `if /I ""&calc&""==""--help"" goto documentation`. Tracing quote state left-to-right: `"`(open)`"`(close) → outside quotes; the following `&` is therefore an unquoted command separator, and `calc.exe` executes. This `if` runs on every invocation before any dispatch. **Not tried:** the actual `cmd.exe` tokenization of the malformed remnant — no Windows interpreter is available in this sandbox, and an Erlang toolchain cannot exercise this axis. Note this is *structurally identical* to `elixir.bat:9-14`, which uses the same `if ""%1""==…` doubled-quote idiom (I read those lines in this pass); the parity is a same-bug-both-files observation, not a mitigation.
- **Delivery note:** From an interactive `cmd` prompt the outer shell consumes a bare `&` before `iex.bat` sees it, so `%1` would be empty. The unquoted metacharacter survives into `%1` when `iex.bat` is started through an argv-array / `CreateProcess` API that does not `cmd`-escape metacharacters — the common way CI systems, editors, and language runtimes (`subprocess`/`child_process`) invoke `.bat` files (the CVE-2024-24576 "BatBadBut" class).
- **Suggested fix:** Replace each comparison with a single-quoted, tilde-dequoted form: `if /I "%~1"=="--help" goto documentation` (and likewise for `-h`, `/h`, `/?`). That keeps any metacharacters inside quotes and still matches a user-quoted flag.

#### `CSCAN-iex-star-args-passthrough`
- **Finding ID:** `CSCAN-iex-star-args-passthrough`
- **Severity:** Medium
- **Confidence:** medium
- **Location:** `iex.bat:29`, `:run` label — `call "%~dp0\elixir.bat" --no-halt --erl "-user elixir" +iex %*`
- **Class:** Command injection (Windows-specific batch argument passthrough)
- **Impact:** Any forwarded argument that reaches `%*` with an unquoted `cmd` metacharacter causes a command break at this `call`, executing an attacker-chosen command *before* `elixir.bat` even parses its arguments. Same privilege and delivery model as the finding above, on the main run path, covering *all* arguments.
- **Evidence:** `asserted`. `%*` is the verbatim argument tail. With an argument `&calc&` surviving unquoted into `%*`, line 29 expands to `call "…\elixir.bat" --no-halt --erl "-user elixir" +iex &calc&`; the unquoted `&` terminates the `call` and runs `calc`. The `"%~dp0\elixir.bat"` target is correctly quoted; the exposure is the trailing `%*`. **Cross-file confirmation (this pass):** I read `elixir.bat` — it does not and cannot neutralise this, because the break happens in `iex.bat`'s own command line before `elixir.bat` is entered. For completeness, `elixir.bat` then re-forwards its own `%*` unquoted into `-extra %*` at line 145 (and echoes it at line 143), so the metacharacter-passthrough class continues one level deeper; that deeper instance belongs to `elixir.bat`'s review, but it means the concern is not absorbed by the delegate. **Not tried:** live `cmd.exe` execution of the `call` re-parse — unavailable in this sandbox.
- **Suggested fix:** In-file mitigation is limited — `call … %*` inherently re-parses; this is the platform-level BatBadBut (CVE-2024-24576) exposure. Durable fixes: ensure callers `cmd`-escape metacharacters when invoking the `.bat`, or prefer a non-`.bat` entry point for programmatic invocation. At minimum, document that `iex.bat` must not be passed untrusted arguments through argv-array APIs without `cmd`-safe escaping.

### Ruled out
- **PATH hijack of the delegated launcher** — line 29 calls `"%~dp0\elixir.bat"`, an explicit script-relative path, not a bare `elixir.bat` resolved through `%PATH%`; PATH-based substitution of the target is not possible. Confirmed this pass by reading the delegate: `elixir.bat` exists as a sibling and is the intended target. Axes examined: PATH resolution of the `call` target, target-path quoting. Not examined: none.
- **Delayed-expansion (`!…!`) injection in this file** — `iex.bat` uses plain `setlocal`, not `setlocal EnableDelayedExpansion`, so injected `!` characters are not re-expanded here under default `cmd` configuration. (Note: `elixir.bat:74` *does* enable delayed expansion for its own parser — a separate file's concern, not inherited by `iex.bat`.) Axes examined: `setlocal` mode in this file. Not examined: none for this file.
- **Temp-file / symlink races (CWE-377)** — no temporary files, `%TEMP%` paths, or create-then-use sequences in this file. Axes examined: filesystem-write sinks. Not examined: none.
- **Privilege / ownership** — no `runas`, setuid-equivalent, `icacls`/`chmod`, or ownership changes. Axes examined: privilege-change sinks. Not examined: none.
- **Download-and-run** — the script fetches nothing; no network or execute-what-you-downloaded path. Axes examined: network/fetch sinks. Not examined: none.
- **`echo` of `%~nx0` (line 15)** — prints only the script's own name to the console; not attacker-controlled in a security-relevant way and not passed to a sink. Axes examined: data flow from `%~nx0` to a sink. Not examined: none.

## Risks / caveats
- **Not executed on-platform.** The environment is Linux with no `cmd.exe`; the Erlang/OTP toolchain present cannot exercise batch parsing. Both findings are static reasoning about `cmd` percent-expansion and command-separator parsing, not observed runs — hence `medium` confidence. A Windows repro (`subprocess.run(["iex.bat","&calc&"])` via an argv API, versus from an interactive prompt) would settle them.
- **Trust-boundary dependence.** For a launcher the invoking user is normally the trust root, in which case "injection" only yields commands the user could already run. Both findings become materially exploitable when `iex.bat` is invoked by a *higher-privileged or automated* context that passes lower-privileged data (a filename from a listing, a name from a manifest) as an argument through an argv-array API — the scenario the severities reflect.
- **`cmd` version/config nuances.** Exact tokenization of malformed `if` statements can vary across Windows `cmd` builds and non-default settings (`cmd /V:ON`, `CMDCMDLINE` quirks); the injection *mechanism* (unquoted metacharacter after percent-expansion) is stable, but the precise remnant command text may differ.

## Did I verify this?
- **`CSCAN-iex-if-quote-injection`** — Not executed. Verified by static trace of `cmd` percent-expansion on lines 8–11; the `cmd.exe` tokenization axis could not be tried (no Windows interpreter in sandbox). Remains `asserted`, medium confidence.
- **`CSCAN-iex-star-args-passthrough`** — Not executed. Static trace of line 29; cross-file leg confirmed by reading `elixir.bat` (delegate does not absorb the break and re-forwards `%*` itself). Live `call` re-parse axis not tried. Remains `asserted`, medium confidence.
- **Ruled-out entries** — the PATH-hijack and delayed-expansion dismissals were tightened this pass by reading `elixir.bat` (delegate confirmed as a script-relative sibling; delayed expansion confirmed scoped to `elixir.bat`, not `iex.bat`). All ruled-out `Not examined` lists are `none` because each concerns a sink class that is simply absent from this file — established by reading the file, no execution required.

## Resolved hedges from first pass
- **"Cross-file parity not audited here / I did not analyze `elixir.bat`'s parser."** Resolved by reading `elixir.bat` in full this pass. Outcome: (1) the `%*` passthrough concern is *not* neutralised by the delegate — the `call`-boundary break in `iex.bat:29` occurs before `elixir.bat` runs, and `elixir.bat` itself re-forwards `%*` unquoted at line 145; (2) `elixir.bat:9-14` uses the same doubled-quote `if ""%1""==…` idiom as `iex.bat:8-11`, so the quote-imbalance pattern is present in both files (relevant to the delegate's own review). This did not add or drop any finding for `iex.bat`; it upgraded the cross-file caveat from "out of scope / assumed" to "read and confirmed."

## Dropped in re-review
None. Both first-pass findings are preserved; no finding was downgraded to removal.
