# /Users/lukas/code/claude-scanner/elixir/bin/elixir.bat
<!-- sha256: 6f4364d64b7b796a84d7533024e0b5694e6471b3b58bc00690ae116cd2bb6e07 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T07:42:30Z_

I've read the full file and its POSIX sibling (`elixir`) for parity context. Here is the report.

## Summary

The Windows launcher faithfully mirrors the POSIX `elixir` parser and quotes user paths correctly, but its `--pipe-to` rejection returns a success exit code and writes to stdout, so tooling cannot detect that the runtime refused to boot.

## Bugfixes

#### `CSCAN-elixir-pipe-to-rejection-exit-zero`
- **Finding ID:** `CSCAN-elixir-pipe-to-rejection-exit-zero`
- **Severity:** Low
- **Confidence:** high
- **Location:** `elixir.bat:132` (parseopts loop, `--pipe-to` arm) → `:end` at line 148
- **Class:** Exit status ignored (error path returns success)
- **Impact:** A Windows user or CI job that invokes `elixir --pipe-to PIPEDIR LOGDIR` (a documented option, explicitly rejected on Windows per the USAGE text at line 63) gets the rejection message printed, the VM never boots — and the process exits `0`. Any wrapper, batch caller, `mix` task, or CI step that branches on the exit status treats the non-boot as success and continues as though Elixir started. The POSIX sibling's equivalent switch-rejection guards (`elixir:185,187`) end with `exit 1`; the Windows path diverges. The message is also written to stdout rather than stderr, so it contaminates any pipeline capturing the launcher's output.
- **Evidence:** `asserted`. Line 132 is `if ""==!par:--pipe-to=! (echo --pipe-to : Option is not supported on Windows && goto end)`. `:end` (line 148) runs only `endlocal` and falls off the end of the file; the last executed command is the successful `echo`, so `%ERRORLEVEL%` is `0` at process exit. (Batch semantics could not be executed to observe — see Risks — but the control flow is unambiguous: there is no `exit /B 1` on this path, unlike the normal run path at line 147 which propagates `erl.exe`'s status.)
- **Suggested fix:** Redirect the message to stderr and exit non-zero, e.g. replace the action with `(echo --pipe-to : Option is not supported on Windows 1>&2 && exit /B 1)`.

## Risks / caveats

- **No Windows/cmd.exe in this sandbox.** The target is a `.bat` file; the environment is Linux with no `cmd.exe` or `wine`, so I could not execute the batch fragments to observe behaviour. Every judgement here is `asserted` from cmd.exe parsing/expansion semantics and from diffing against the POSIX `elixir` script, not from a run. A validator with a Windows host should confirm the `--pipe-to` exit code.
- **The `""==!par:FLAG=!` idiom is safer than it looks.** Because `par` is wrapped in quotes at line 100 (`set par="!par:"=\"!"`) and the substitution result is compared against the literal empty-quoted string `""`, the substring test only fires when removing the flag leaves *nothing but the two wrapping quotes* — i.e. it is effectively an exact match. That is why `--boot` does not shadow `--boot-var`, `--erl` does not shadow `--erl-config`, and `--color` does not swallow `--no-color`. I could not run cmd to confirm, but the anchoring is structural.
- **`%*` re-passes the full original argv** (line 145), so distribution/release options such as `--cookie`/`--name`/`--boot` reach Elixir's plain-argument list *after* `-extra`, whereas the POSIX script strips them. This is a deliberate, long-shipped property of the Windows launcher (it never rebuilds the arg vector), and Elixir's own CLI parser is expected to tolerate the duplication; I did not treat it as a defect.

### Ruled out
- **`--logger-otp-reports` / `--logger-sasl-reports` pass the value unvalidated (`elixir.bat:126-127`) while the sibling only forwards `true`/`false` (`elixir:135,141`).** The divergence is real but is only reachable by passing a value that violates the documented `BOOL` contract; with the in-contract values `true`/`false` the Windows path produces valid `-logger handle_otp_reports true` output. Out of scope (contract-violating argument).
- **`%1` used instead of `%~1` on the logger lines.** For the documented `BOOL` value (`true`/`false`, no surrounding quotes) `%1` and `%~1` are identical, so no in-contract failure results.
- **`-noshell` is added even in `+iex` mode (line 140).** Matches the POSIX script (`elixir:229` adds `-noshell` unconditionally); iex installs its own user I/O server, so this is intended, not a dropped-shell bug.
- **`-S`, `--short-version`-with-trailing-args, `-v`, `--version` are not matched in the parse loop and fall through to `:run`.** This mirrors the POSIX `*)` catch-all/`break`; `%*` forwards them to Elixir, which handles them. No divergence.
- **Install path containing a space (e.g. `C:\Program Files\elixir`).** `SCRIPT_PATH` (`%~dp0`) is quoted everywhere it is consumed (`-elixir_root "%SCRIPT_PATH%..\lib"`, `-pa "%SCRIPT_PATH%..\lib\elixir\ebin"`, `"%ERTS_BIN%erl.exe"` at lines 140/143/145), and `--cookie`/`--name`/`--boot` values are quoted into `parsErlang`. Spaces are preserved; not a defect.

