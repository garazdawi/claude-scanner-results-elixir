# Threat Model — `elixir/bin` (Elixir/Erlang CLI launcher scripts)

## Protocol / standard implemented
This folder contains **no `.erl` files**. It is the Elixir toolchain's **command-line launcher suite** — the executables a user or release invokes to boot the BEAM (`erl`/`erl.exe`/`run_erl`). There is no wire protocol or RFC here. The relevant "standards" are shell semantics: **POSIX Shell Command Language (IEEE Std 1003.1)** for `elixir`/`iex`/`elixirc`, Windows **cmd.exe batch** parsing for `*.bat`, and **PowerShell** argument arrays for `mix.ps1`. `mix` is an Elixir script (`#!/usr/bin/env elixir`) that just calls `Mix.CLI.main()`.

Public entry points (what a caller invokes):
- `elixir` / `elixir.bat` — argument parser + `exec erl … -extra "$@"`; the core of the suite.
- `iex` / `iex.bat` — thin wrapper: `exec elixir --no-halt --erl "-user elixir" +iex "$@"`.
- `elixirc` / `elixirc.bat` — wrapper: `exec elixir +elixirc "$@"`.
- `mix`, `mix.bat`, `mix.ps1` — Mix task runner front-ends.

## Trust / contract boundaries
The dominant boundary is **operator/local-user-supplied argv and environment**, not a network peer. Data crossing into these scripts:
- **argv** — fully caller-controlled. The parser routes flags into two sinks: `erl_set`/`ERL` (translated to `erl` switches like `-setcookie`, `-config`, `-boot`, `-args_file`, `-name`) and the `-extra` passthrough that becomes the Elixir program's own argv. `-e`/`--eval`/`--rpc-eval` carry **arbitrary code strings** to evaluate — by design.
- **Environment** — `ELIXIR_ERL_OPTIONS` and (Windows) `ERL_COMPILER_OPTIONS` are spliced **unquoted** into the final command line (`$ELIXIR_ERL_OPTIONS $ERL "$@"`, line 229); `ERTS_BIN` selects the `erl` binary; `OSTYPE` triggers `cygpath`. Anyone who controls the environment controls what BEAM boots.
- **Filesystem** — `readlink_f` follows symlinks from `$0` to locate `SCRIPT_PATH`, then trusts `../lib/elixir/ebin` for the code path. `--erl-config`, `--vm-args`, `--boot` name files read by the VM.

Implicit contract the caller must honour: these are **trusted-invocation** tools. There is no `-spec`; the design assumes whoever runs `elixir` already has the privilege to run arbitrary code. Values like PIPEDIR/LOGDIR are only checked for a leading `-` (switch-injection guard, lines 184-188), nothing more.

## Attacker model
Primarily a **local/operator context**, not a remote peer:
- **In scope (bugs, not vuln classes):** correctness of the hand-rolled argument shifter — miscounting `S`/`C` (shift vs copy) so a flag's argument leaks into `-extra` or a following flag is swallowed; quoting/escaping so an argument containing spaces, quotes, or shell metacharacters is split or re-interpreted before reaching `erl`; the `--pipe-to` `sed` escaper (line 234) that must neutralise every metacharacter it doesn't whitelist.
- **Windows-specific:** `.bat` files build command lines by textual substitution (`!par:"=\"!`, `%~1`, delayed expansion) — classic **batch argument-injection / quote-breakout** territory. `mix.bat` forwards `%*` verbatim.
- **Out of scope:** these scripts do not authenticate anyone. `--cookie`/`--name`/`--sname` configure **distributed Erlang** — the actual auth boundary (cookie-based node auth, epmd) lives in the runtime, not here. A malicious operator already wins; the interesting flaw is one where *lower-privileged* input (an env var, a filename, a script argument passed through by a higher-privileged wrapper) reaches a sink it shouldn't.

## Common attack patterns for this code class
- **Argument/switch injection** — a value beginning with `-` sneaking into `erl`'s own switch position (the `--pipe-to` guard exists precisely for this; check every other passthrough).
- **Word-splitting / glob expansion** on unquoted `$ERL`, `$ELIXIR_ERL_OPTIONS`, `$ESCAPED` — an operator-set env var with spaces changes the boot invocation.
- **Distributed-Erlang cookie exposure** — `-setcookie` on the command line is visible in `ps`; relevant if a wrapper embeds a secret.
- **Symlink / PATH manipulation** of `SCRIPT_PATH` → loading attacker-controlled `ebin`/`.boot`/`-config`.
- **Batch delayed-expansion breakout** (`!…!`) and PowerShell array-flattening (`mix.ps1` reintroducing commas) mangling arguments.

## Supervisor / process structure
None in this folder. There are **no gen_servers, gen_statems, or supervisors** — these are `exec`-and-replace launcher scripts. The only process nuance: `--pipe-to` re-execs via `run_erl -daemon` (detached), and `iex` sets `-user elixir`. Process-safety boundaries live in the OTP libraries these scripts *start*, not here.

## Cross-file invariants
- `iex`, `elixirc`, and (via shebang) `mix` **all delegate to `elixir`** — the argument parser in `elixir`/`elixir.bat` is the single choke point; a quoting bug there is inherited by every other entry point. Review `elixir` first and treat the wrappers as thin.
- `elixir.bat` ↔ `elixir` (sh) must keep **parity**: the same flags, same shift-counts, same escaping semantics across two hand-written parsers. Divergence (e.g. a flag handled in one but passed through in the other) is a likely defect and only visible when both files are read together.
- `mix.ps1` assumes `mix.bat` sits beside it and that its comma-rejoining exactly reconstructs what `mix.bat`/`elixir.bat` expect — a cross-file argv-encoding contract.
- All scripts assume `SCRIPT_PATH/../lib/elixir/ebin` is trustworthy; none re-validate it. A finding about loading untrusted code from that path cannot be dismissed by appeal to another file in this folder — nothing here checks it.