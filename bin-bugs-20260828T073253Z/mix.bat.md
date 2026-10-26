# /Users/lukas/code/claude-scanner/elixir/bin/mix.bat
<!-- sha256: b3f6244f32180e4b0fcfc091b95c814c48f3ba7529029985c2a85d085d9b16f2 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T07:41:50Z_

## Summary

`mix.bat` is a correct one-line delegator that quotes both path arguments and forwards `%*` verbatim to `elixir.bat`, and I found no defect reachable through its documented `mix [task] [args]` invocation.

## Bugfixes

No findings

## Risks / caveats

- **Could not execute the batch file.** The sandbox is Linux; there is no `cmd.exe`, so every judgment below about `%~dp0`, `%*`, and `call` errorlevel propagation is `asserted` (reasoned against documented cmd.exe semantics), not `verified` by running. A Windows shell is required to confirm the path-normalization and quoting claims empirically.
- The whole-flow reasoning depends on `elixir.bat`'s parser: because the first positional (`"%~dp0\mix"`) matches no switch arm, elixir.bat's loop falls straight to `:run` and every user argument reaches Mix through `-extra %*` unparsed. That is the intended contract; if `elixir.bat`'s parse loop were changed to consume/reorder leading positionals, `mix.bat` would silently break — a cross-file invariant, not a `mix.bat` line.

### Ruled out

- **Double backslash in `"%~dp0\mix"` and `"%~dp0\elixir.bat"`** — `%~dp0` always ends in `\`, so these expand to `...\bin\\mix` / `...\bin\\elixir.bat`. Not a bug: Win32 path canonicalization (`RtlGetFullPathName`) collapses consecutive separators in the middle of a path before the file is opened, and this exact doubled-separator form is the shared convention across all four launchers (`elixirc.bat:39`, `iex.bat:29` both use `"%~dp0\elixir.bat"`). Named mechanism + named parity source, so this stands as a rule-out rather than an untested assertion — modulo the Windows-execution caveat above.
- **Space in the install path (e.g. `C:\Program Files\Elixir\bin`)** — the classic High launcher bug. Both forwarded paths are double-quoted here, and `elixir.bat` re-quotes them onward (`set "par=%~1"`, `-extra %*`, `-elixir_root "%SCRIPT_PATH%..\lib"`), so the space survives to `erl.exe`. No word-splitting is introduced by `mix.bat`.
- **Exit status swallowed** — `mix.bat`'s only executed statement is the trailing `call`; `elixir.bat` ends with `exit /B %ERRORLEVEL%`, and `call` preserves that errorlevel as `mix.bat`'s own exit code with nothing after it to overwrite it. No masking.
- **`%*` argument mangling / injection** — `%*` reproduces the original command tail verbatim (including a user's own quotes and metacharacters); `mix.bat` neither re-parses nor re-splits it, so args like `mix run "a & b"` pass through unchanged. Standard, correct behaviour identical to the sibling launchers.
- **Environment leakage / missing `setlocal`** — `mix.bat` sets no variables of its own, so the absence of `setlocal` leaks nothing; the callee `elixir.bat` manages its own scope.

