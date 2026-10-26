# /Users/lukas/code/claude-scanner/elixir/bin/mix.bat
<!-- sha256: b3f6244f32180e4b0fcfc091b95c814c48f3ba7529029985c2a85d085d9b16f2 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T20:08:54Z_

I've now traced the full path: `mix.bat` forwards `%*` into `elixir.bat`, and `elixir.bat`'s parse loop (lines 93–132) stops at the first non-option token — which is always the `mix` script path that `mix.bat` inserts as the first argument. That determines whether forwarded args can reach the `erl` switch sinks. Here is my report.

---

# Security review — `elixir/bin/mix.bat`

## Summary
A 7-line cmd.exe batch wrapper whose only action is `call "%~dp0\elixir.bat" "%~dp0\mix" %*`; the sole security-relevant surface is the verbatim `%*` argument forwarding, which is the standard idiom but carries the inherent Windows batch re-parsing (BatBadBut) exposure when a caller passes untrusted, unquoted metacharacters.

## Security

#### `CSCAN-mix-batch-argv-cmd-injection`
- **Finding ID:** `CSCAN-mix-batch-argv-cmd-injection`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `elixir/bin/mix.bat:7` — top-level `call` statement
- **Class:** Command injection (batch `%*` re-expansion) / Windows-specific
- **Impact:** If `mix.bat` is invoked programmatically by a higher-trust wrapper (an IDE plugin, build tool, or language runtime's process-spawn API) with an attacker-influenced argument — e.g. a filename or task name — that argument can reach `%*` as an *unquoted* command tail containing a cmd operator (`&`, `|`, `<`, `>`, `(`, `)`). When line 7 expands `%*` and `call` re-parses the resulting line, that operator is interpreted as a command separator, yielding arbitrary command execution in the caller's context. This is the CVE-2024-24576 ("BatBadBut") class.
- **Evidence:** `asserted`. The sandbox is Linux with no `cmd.exe`, so batch parsing cannot be executed here. Mechanism (well-documented cmd semantics): `%*` reproduces the raw argument tail verbatim, including characters that arrived unquoted (e.g. via a parent caret-escape `mix.bat foo ^& calc`, or via a runtime that escapes an argument for a normal `.exe` rather than for batch). Line 7 splices that tail into `call "…\elixir.bat" "…\mix" foo & calc`; the `&`, no longer escaped at this level, terminates the `call` and runs the following text as a separate command. `mix.bat` does not enable delayed expansion, so `!…!` is *not* an additional vector here, but `& | < > ( )` in the re-parsed line are.
- **Suggested fix:** There is no complete in-file remedy — `%*` verbatim forwarding is the correct and least-bad idiom, and any metacharacter filter would corrupt legitimate task arguments. The durable fix is caller-side: wrappers that hand untrusted data to `mix.bat` must quote each argument for cmd/batch specifically (not merely for a normal executable). Document this expectation for the `.bat` entry points; treat it as a known platform limitation of invoking `.bat` files with untrusted argv.

### Ruled out
- **Argument injection into `erl`/Elixir switches via forwarded args** — *not present*. `mix.bat` inserts the `mix` script path (`"%~dp0\mix"`) as the **first** argument to `elixir.bat`, ahead of `%*`. `elixir.bat`'s parse loop (`:startloop`, lines 93–132) advances only on recognized `+…`/`-…`/`--…` tokens (each branch does `goto startloop`); the first non-option token — the `mix` path — matches none of them and falls straight through to `:run` (line 134). All user-supplied args therefore land in `%*` at line 145 as `-extra` payload (Elixir `System.argv`) and never occupy an `erl`/Elixir switch position such as `-setcookie`, `-config`, `-boot`, or `-args_file`. Basis: direct reading of `elixir.bat`'s parser (could not be executed — no `cmd.exe` in-sandbox), lines cited.
- **PATH hijack of `elixir.bat`** — *not present*. Line 7 invokes the wrapper by an explicit, quoted path (`"%~dp0\elixir.bat"`), not a bare name, so `%PATH%`/`%PATHEXT%` resolution is not consulted and a planted `elixir.bat` elsewhere on PATH is not reached.
- **Quote-breakout / metacharacter injection via `%~dp0`** — *not present*. `%~dp0` (the script's own directory) is used only inside double quotes (`"%~dp0\elixir.bat"`, `"%~dp0\mix"`). Windows path names cannot contain `"`, so the install path cannot close the quotes, and a `&`/space in the directory name stays contained within the quotes.
- **Delayed-expansion (`!VAR!`) breakout** — *not present in this file*. `mix.bat` never issues `setlocal enabledelayedexpansion`, so `!` in an argument is literal here. (Delayed expansion is enabled inside `elixir.bat` at line 74 — that is `elixir.bat`'s concern, not `mix.bat`'s.)
- **Malicious `elixir.bat`/`mix` planted in `%~dp0`** — *not a distinct exposure*. `%~dp0` is `mix.bat`'s own install directory; an attacker able to write there could replace `mix.bat` itself, so no privilege boundary is added by trusting sibling files in the same directory.
- **Env / PATH / temp-file / privilege / download-and-run classes** — *absent*. The file sets no environment variables, uses no `setlocal`, creates no temp files, invokes no `sudo`/`chmod`/setuid path, and fetches nothing from the network.

## Risks / caveats
- **No `cmd.exe` in the sandbox.** The host is Linux, so every batch-semantics claim above (both the reported finding and the rule-outs) is reasoned from documented cmd.exe behavior and from reading `elixir.bat`, not observed by execution; they are marked `asserted`. A confirmatory run on Windows (`mix.bat foo ^& calc` for the finding; `set ELIXIR_CLI_DRY_RUN=1 & mix.bat --erl "x"` to confirm forwarded switches reach `-extra` rather than `erl`) would settle them.
- **Cross-file dependency.** The safety of `%*` forwarding relies on `elixir.bat` continuing to stop option parsing at the first positional token. If a future `elixir.bat` change scans all tokens for switches, forwarded `mix` arguments could begin reaching `erl` switch sinks — the rule-out above would then no longer hold. This is a parity/invariant risk between the two hand-written parsers, not a defect in `mix.bat` as it stands.
- The reported finding's real-world severity is deployment-dependent: Low when a developer invokes `mix` directly (they are the trust root), potentially higher where tooling shells out to `mix.bat` with project- or attacker-derived strings. It is rated for this file in isolation.

