# /Users/lukas/code/claude-scanner/elixir/bin/mix
<!-- sha256: 0a48d65a9980f2b923bba597e0ebdde3049e3271089ffffb9265d5cf7954b659 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T20:06:00Z_

# Security Review — `elixir/bin/mix`

## Summary
This file is a 7-line Elixir launcher (`#!/usr/bin/env elixir` → `Mix.CLI.main()`) with no shell, no data handling, and no sink in scope, so it contains no security defect of its own.

## Security

No findings.

The file, in full, is:

```
#!/usr/bin/env elixir

# SPDX-License-Identifier: Apache-2.0
# SPDX-FileCopyrightText: 2021 The Elixir Team
# SPDX-FileCopyrightText: 2012 Plataformatec

Mix.CLI.main()
```

Interpreter it actually runs under: **not a POSIX shell at all** — it is executed by the Elixir script runner, located via `/usr/bin/env elixir`. There is no `sh`/`dash`/`bash` parsing of this file, so the entire shell-quoting / word-splitting / `set -euo pipefail` class is inapplicable. The single statement `Mix.CLI.main()` is called with no arguments; all of Mix's real argument, environment, and filesystem handling occurs inside the compiled Elixir/OTP library `Mix.CLI`, which is not this file and is out of scope for this review (verified by grep: the file contains no `eval`, `exec`, backticks, `$(...)`, `System.*`, `:os.*`, `Port.*`, `argv`, or `getenv` — the only match was the word `env` in the shebang).

### Ruled out

- **Interpreter resolution via `#!/usr/bin/env elixir` (PATH search).** `env` resolves `elixir` through the caller's `$PATH`, so whichever `elixir` appears first runs. This is the file's documented, intended invocation mechanism (deliberate, so version managers such as `asdf`/`kiex` select the active Elixir), and the invoking user — who owns their own `$PATH` — is the trust root; the threat model explicitly treats these as trusted-invocation tools where "a malicious operator already wins." Nothing in this file establishes a privilege boundary in which *lower-privileged* input controls `$PATH` for a *higher-privileged* execution (no `sudo`, setuid, or privileged-wrapper context appears here — verified against the complete 7-line file). Flagging this would be flagging the `/usr/bin/env` pattern itself, not a defect specific to this launcher. Noted as a residual concern below rather than dropped silently.
- **Argument injection / word splitting / `-`-leading argv.** `Mix.CLI.main()` receives no arguments from this file; there is no argv assembly, no unquoted expansion, and no passthrough here. The choke point for argv quoting is `elixir`/`elixir.bat` (a separate file), not this one — confirmed by full read: no argument handling exists in these 7 lines.
- **Command injection (`eval`, `sh -c`, `$(...)`, backticks).** None present — verified by grep over the whole file.
- **Environment trust (`LD_PRELOAD`, `IFS`, `ERL_*`, `BASH_ENV`, etc.).** The file reads and splices no environment variable; any env handling is internal to `Mix.CLI`.
- **Temp files / predictable paths, privilege/ownership, download-and-run.** No filesystem writes, no `/tmp`, no `$$`, no `sudo`/`chmod`, and no network fetch appear anywhere in the file — verified by complete read.
- **Windows-specific batch/PowerShell issues.** N/A — this is the Unix launcher; `mix.bat`/`mix.ps1` are separate files.

## Risks / caveats
- **Scope boundary:** the substantive attack surface for the `mix` entry point (argument parsing, quoting, env-var splicing, `-config`/`-boot` handling) lives in the `elixir` sh script and `elixir.bat` that this launcher delegates to via its shebang, and in the compiled `Mix.CLI` module — none of which is this file. A clean result here says nothing about those; they must be reviewed on their own.
- **Residual `env`/PATH note:** the `/usr/bin/env elixir` shebang is safe under normal user invocation but would resolve an attacker-planted `elixir` first if `mix` were ever invoked by a higher-privileged process with an attacker-influenced `$PATH` (e.g. a privileged CI step or wrapper that fails to sanitize `PATH`). That is a property of the caller's environment, not a defect in this file, and is out of scope per the folder's trusted-invocation threat model — but worth keeping in mind for any wrapper that runs `mix` across a privilege boundary.
- I did not execute `Mix.CLI.main()` (no Elixir runtime is required to assess this file, and its logic is out of scope); the conclusions rest on a complete read of all 7 lines plus a grep for dangerous constructs, both shown above.

