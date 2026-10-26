# Security Scan Summary — `elixir/bin` launcher suite

**Scope:** 9 CLI launcher scripts that boot the BEAM — the POSIX-sh trio (`elixir`, `iex`, `elixirc`), the Windows batch trio (`elixir.bat`, `iex.bat`, `elixirc.bat`), the Mix front-ends (`mix`, `mix.bat`, `mix.ps1`). Per the threat model these are *trusted-invocation* tools: the invoking user is normally the trust root, so the findings that matter are those where **lower-trust input** (an inherited env var, a filename, or an argument forwarded by a higher-privileged wrapper/CI step) reaches an execution sink.

**Evidence caveat:** the scan ran on aarch64 Linux with **no `cmd.exe`/`wine`/PowerShell**. Every **sh** finding below was **verified by execution**; every **Windows batch/PowerShell** finding is **`asserted`** (reasoned from documented cmd.exe semantics, not run) and should be confirmed on Windows. No independent `VALIDATION.md` pass was present.

---

## Ranked findings

### 1. `--pipe-to`/`run_erl` escaper passes newlines raw → shell command injection (RCE)
**High · high confidence · VERIFIED · `elixir:234` (reachable via `iex`, `elixirc`)**\
In `--pipe-to` (detached-daemon) mode the assembled command is handed to `run_erl`, which runs it through `/bin/sh`. The escaper backslash-escapes every byte outside `[a-zA-Z0-9_/-]` — but a **newline never matches** and passes through raw, so a newline in any argument value (untrusted program data after `-extra`, or a `--cookie`/`--erl-config`/`--boot`/`--vm-args`/`--name` value sourced from a file/env) splits the command line and starts a second command. Confirmed by execution: `ELIXIR_CLI_DRY_RUN=1` emits an unescaped `\n` inside the single `run_erl` argument (`od -c`). The escaper exists precisely to make these values safe and fails open on a common metacharacter. `iex` and `elixirc` forward `"$@"` verbatim, so all three sh entry points inherit it. **Fix:** single-quote-wrap each part, or reject newline-bearing values.

### 2. `CDPATH`-poisoned `readlink_f` redirects `SCRIPT_PATH` → execs attacker's `elixir` (RCE)
**Medium · high confidence · VERIFIED (dash + bash) · `iex`, `elixirc`, and — unowned — `elixir`**\
`readlink_f` runs `cd "$(dirname "$0")"` on a relative invocation path (e.g. the very common `bin/iex`) **without scrubbing `CDPATH`**. An attacker-influenced `CDPATH` (inherited from CI vars, direnv, Docker `ENV`, systemd `Environment=`) silently redirects `cd bin` into `<cdpath>/bin`, so `SCRIPT_PATH` becomes attacker-controlled and the launcher `exec`s an attacker-planted `elixir` binary — defeating the author's own "run by absolute path, not PATH" safeguard. The `>/dev/null` on the `cd` hides the redirect. Reproduced end-to-end for `iex` ([`CSCAN-iex-cdpath-hijack`](./iex.md#cscan-iex-cdpath-hijack)) and `elixirc` ([`CSCAN-elixirc-cdpath-cd-hijack`](./elixirc.md#cscan-elixirc-cdpath-cd-hijack)).
> **Cross-file gap ([`CSCAN-crossfile-elixir-cdpath-hijack-untracked`](./CROSS_FILE.md#cscan-crossfile-elixir-cdpath-hijack-untracked)):** the core `elixir` launcher — the delegate that `iex`/`elixirc`/`mix` all `exec` — carries a **byte-identical** `readlink_f` (`elixir:69-77`, `exec` at `219`/`245`), so the *verified* exploit runs unchanged against it. Yet **no report owns this**: `iex`/`elixirc` both defer it as "tracked in `elixir`," while `elixir.md` files no CDPATH finding and dismisses the nearest concern as "already-privileged." The central choke point is the unowned instance. **Fix:** `unset CDPATH` (or `CDPATH= cd …`) in **all three** sh scripts — most importantly `elixir`.

### 3. `iex.bat` first-argument quote-imbalance → cmd command injection
**High · medium confidence · asserted · `iex.bat:8-11` (same idiom in `elixir.bat:9-14`)**\
The help check `if /I ""%1""==""--help""` places two literal quotes before `%1`, so after percent-expansion the argument begins in an **unquoted** parse context: `%1 = &calc&` expands to a line where `&` is an unquoted command separator and `calc.exe` runs. This executes on **every invocation** before any dispatch. Not exploitable from an interactive prompt (the outer shell eats a bare `&`), but the metacharacter survives when `.bat` is launched via an argv-array/`CreateProcess` API — the common CI / editor / `subprocess` path (CVE-2024-24576 "BatBadBut" class). The identical doubled-quote idiom sits in `elixir.bat:9-14`. **Fix:** `if /I "%~1"=="--help" …` (single-quoted, tilde-dequoted).

### 4. Windows loses the sh allow-list on `--logger-otp/sasl-reports` → unchecked value into the injectable re-parse
**Medium · high confidence · cross-file cluster ([`CSCAN-cluster-logger-report-validation-split`](./CLUSTER_root-0.md#cscan-cluster-logger-report-validation-split)) · `elixir` sh 133-144 vs `elixir.bat:126-127`, sink `:145`**\
The POSIX `elixir` restricts these two flags' values to `{true,false}` **before** they reach the command string (anything else is consumed but spliced nowhere). `elixir.bat` **drops the allow-list** and splices bare `%1` straight into `parsErlang`, which cmd.exe re-parses at the shared sink `"%ERTS_BIN%erl.exe" … %parsErlang% … -extra %*` (`:145`). So `--logger-otp-reports <anything>` reaches `erl` unchecked (invalid logger config / boot misbehaviour), and a value carrying cmd metacharacters rides the same injectable re-parse. Invisible to either single-file pass — the *intent* to constrain to booleans only exists in the sh sibling. **Fix:** mirror the `{true,false}` gate in the batch parser and consume via delayed expansion.

### 5. `ELIXIR_ERL_OPTIONS` spliced into the cmd re-parse → arbitrary Windows command execution
**Medium · medium confidence · asserted · `elixir.bat:145` (`disabledelayedexpansion` at `:135`)**\
`%ELIXIR_ERL_OPTIONS%` is substituted into the command *text* while the `( … )` block is parsed, then executed, so a `&`/`|`/`>` in the env var (`foo & malware.exe`) runs as a Windows command — an escalation beyond the documented "pass extra `erl` switches." The dry-run branch offers no protection (the `echo` is re-parsed too). The POSIX sibling is **not** injectable (unquoted `$ELIXIR_ERL_OPTIONS` only word-splits into literal argv tokens), confirming this is a genuine sh-vs-bat divergence. **Fix:** reference via delayed expansion (`!ELIXIR_ERL_OPTIONS!`) at the exec line, or reject metacharacters up front.

### 6. "BatBadBut" verbatim `%*`/argv passthrough into cmd.exe re-parse (family, 4 files)
**Medium–Low · medium confidence · asserted · caller-side fix**\
The Windows/PowerShell wrappers forward caller argv verbatim into a cmd.exe re-parse chain; an unquoted `&`/`|`/`<`/`>` in an argument forwarded by a higher-trust wrapper executes. No complete in-file remedy exists (metacharacter filtering corrupts legitimate args) — the durable fix is caller-side cmd-safe quoting, plus documenting that these `.bat`/`.ps1` entry points must not receive untrusted argv.
- [`CSCAN-iex-star-args-passthrough`](./iex.bat.md#cscan-iex-star-args-passthrough) (Medium) — `iex.bat:29` `call … +iex %*`; breaks *before* `elixir.bat` runs.
- [`CSCAN-mix-ps1-batch-arg-injection`](./mix.ps1.md#cscan-mix-ps1-batch-arg-injection) (Medium; sink verified by read) — `mix.ps1:27` `& $mixBatPath $newArgs`; a no-whitespace value like `x&calc` is passed unquoted across the PowerShell→cmd boundary.
- [`CSCAN-elixir-bat-argv-value-cmd-injection`](./elixir.bat.md#cscan-elixir-bat-argv-value-cmd-injection) (Low) — `elixir.bat:131` splices `--erl` **unquoted**; an embedded `"` in `--cookie a"&calc` breaks the `"%~1"` quoting into the `:145` re-parse.
- [`CSCAN-mix-batch-argv-cmd-injection`](./mix.bat.md#cscan-mix-batch-argv-cmd-injection) (Low) — `mix.bat:7` `call … %*`; the plain verbatim-forward idiom.

### 7. Windows re-delivers consumed distribution/boot switches (incl. `--cookie`) into the program's argv
**Low · medium confidence · cross-file cluster ([`CSCAN-cluster-consumed-switch-argv-leak`](./CLUSTER_root-0.md#cscan-cluster-consumed-switch-argv-leak)) · `elixir` sh vs `elixir.bat:145`**\
The sh `elixir` fully consumes and strips `--cookie`/`--name`/`--boot`/`--erl-config`/`--vm-args`/`--hidden` from the program passthrough (they go before `-extra`). `elixir.bat` re-appends the **entire original tail** via `-extra %*` (unaffected by `shift`), so on Windows the distributed-Erlang **auth cookie** and boot-file paths are *also* handed to the Elixir program as ordinary `System.argv/0` — leaking via any code that logs/echoes/crash-dumps its argv. **Fix:** build an explicit post-`-extra` list of only program args, as sh does.

### 8. `elixir.bat` runs bare `erl.exe` → current-directory search
**Low · medium confidence · asserted · `elixir.bat:89-90 → :145`**\
With `ERTS_BIN` forced empty, line 145 invokes the bare name `"erl.exe"`; cmd.exe searches the **CWD before PATH**, so a planted `erl.exe` in the launch directory runs as the victim. Unix resolves `erl` via PATH only. Release builds template `ERTS_BIN` to an absolute path (removing this); the exposure is for source/PATH-based installs. **Fix:** invoke `erl.exe` by fully-qualified path.

### 9. Dead `--pipe-to` switch-injection guard
**Low · high confidence · VERIFIED · `elixir:184-188`**\
The guard meant to reject a `-`-leading PIPEDIR/LOGDIR captures the **stdout** of `starts_with` (which signals via **exit status**, printing nothing): `[ "$(starts_with …)" ]` is always `[ "" ]`, always false, so the guard **never fires**. Damage is contained (values are later quoted/positional; worst case is `mkdir` option-injection aborting under `set -e`) but it is a security check that does nothing. **Fix:** branch on exit status (`if starts_with … "-"; then`).

---

## De-ranked / non-findings (recorded for completeness)

- **[`CSCAN-elixirc-for-loop-prescan-parse`](./elixirc.bat.md#cscan-elixirc-for-loop-prescan-parse-downgraded-from-cscan-elixirc-for-loop-quote-injection) — downgraded, near-dropped (Low · low confidence).** The first pass's Medium `" & calc & "` → `calc` command-injection PoC against `elixirc.bat:9-15` was **withdrawn**: non-`/F` `FOR` splits on whitespace (so the token stream was wrong) and plain `%%A` references aren't re-scanned for `&`/`|`. Only a quote-balance parse-robustness concern survives; needs Windows confirmation before it counts as anything.
- **[`CSCAN-crossfile-mix-readlink-parity-overclaim`](./CROSS_FILE.md#cscan-crossfile-mix-readlink-parity-overclaim) — Contradicted (Low, remediation hygiene, not a vuln).** `elixirc.md` claims the `readlink_f`/CDPATH fix should also land in `mix`, but `mix` is a 7-line Elixir script (`#!/usr/bin/env elixir` → `Mix.CLI.main()`) with **no `readlink_f`/`cd`/CDPATH**. Correct the parity scope to `{elixir, iex, elixirc}` (+ the Windows chain); no source change in `mix`.
- **`mix` (sh) — no findings.** 7-line launcher, no shell parsing of its own; attack surface lives in `elixir`/`elixir.bat` and the compiled `Mix.CLI`.
- **Coverage gap — Unresolved ledger row:** `elixir.bat` ↔ `elixir` (sh) **shift-count / flag-by-flag parity** was never audited by any report. Not a defect, but the one cross-file invariant the threat model flags that remains unverified — a divergence there (a flag consumed in one parser, passed through in the other) would be a likely defect and only visible by reading both together.
