# Cross-file findings

## Asserted-dependency ledger

These reports do not use the literal `* Depends on:` syntax; the cross-file contracts are stated in prose ("Depends on" / "verified by reading the sibling" / "tracked there" / cross-file caveats). Every such asserted dependency is enumerated below.

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| elixir.md | `iex`/`elixirc` forward `"$@"` verbatim → `--pipe-to` escaper reachable through them | Confirmed | iex.md:24/71 (`iex:38` exec) and elixirc.md:32 (`elixirc:43` exec) confirm verbatim forwarding. |
| elixir.md | No in-folder caller lowers argv trust while preserving `ELIXIR_ERL_OPTIONS` (env-trust negative) | Confirmed | elixir.md:70-75 read iex/elixirc/mix; mix.md confirms mix does no shell env/argv handling. |
| elixir.md | `SCRIPT_PATH/../lib/elixir/ebin` not re-validated by any caller | Confirmed | Consistent with THREAT_MODEL:40 standing invariant; iex/elixirc derive SCRIPT_PATH identically. |
| elixir.md | `eval` in `erl_set` performs indirect assignment, no re-parse | Confirmed | elixir.md:54 verified `$(PWNED=yes)` stored literally, did not execute. |
| iex.md | Sibling `elixir` carries byte-identical `readlink_f` → identical CDPATH exposure, **"tracked there"** in elixir | **Contradicted** | elixir.md reports no CDPATH finding and rules out the SCRIPT_PATH concern as "already-privileged" (elixir.md:58). See finding below. |
| elixirc.md | `elixir` `readlink_f` byte-identical (elixir:69-77) → shared CDPATH/CDPATH-`cd` defect | Confirmed (code identity) | Matches iex.md:34/48 diff-IDENTICAL; the shared code exists. (That elixir *reports* it is the contradiction above.) |
| elixirc.md | `iex` shares the CDPATH defect (parity "stated, not verified") | Confirmed | iex.md independently verifies `CSCAN-iex-cdpath-hijack` in `iex`. |
| elixirc.md | `mix` shares the `readlink_f`/CDPATH defect; fix applies "across elixir, iex, and mix" | **Contradicted** | mix.md:28 — `mix` is an Elixir script (`#!/usr/bin/env elixir`), 7 lines, no `readlink_f`/`cd`/CDPATH. See finding below. |
| elixirc.md | `+elixirc` consumed as a mode marker by elixir's parser (elixir:113-115), not a passthrough sink | Confirmed | elixir.md characterises `elixir` as the argv parser; elixirc.md:35 confirmed by reading `elixir.bat:103` sibling. |
| mix.ps1.md | `mix.bat` forwards `%*` verbatim (`mix.bat:7`) | Confirmed | mix.bat.md:14 confirms `call "%~dp0\elixir.bat" "%~dp0\mix" %*`; mix.ps1.md:40 verified by read. |
| mix.ps1.md | `elixir.bat` re-reads `%1`/`%2` positionally with no sanitisation (chain not neutralised) | Confirmed | mix.ps1.md:41 read `elixir.bat:1-40`; consistent with elixir.bat.md. |
| iex.bat.md | `elixir.bat` re-forwards `%*` unquoted at line 145 | Confirmed | elixir.bat.md:56 documents the `%*`-after-`-extra` sink at line 145. |
| iex.bat.md | `elixir.bat` uses the same doubled-quote `if ""%1""==` idiom (line 9-14) | Confirmed | elixir.bat.md / THREAT_MODEL note the same quote-imbalance pattern. |
| mix.bat.md | `%*` forwarded through `elixir.bat` to `erl.exe` (BatBadBut chain) | Confirmed | elixir.bat.md:56 confirms downstream unquoted `%*` re-expansion. |
| elixir.bat.md | Sibling `elixir` (sh) splices `$ELIXIR_ERL_OPTIONS` unquoted at line 229; sh version *not* cmd-injectable | Confirmed | elixir.md:52 verified word-splitting/globbing occurs in sh but no command executes (literal argv tokens). |
| THREAT_MODEL:39 | `mix.ps1` comma-rejoining reconstructs what `mix.bat`/`elixir.bat` expect (argv-encoding contract) | Confirmed (addressed, non-security) | mix.ps1.md:28 traces the comma-split mangling as a correctness bug that introduces no new metacharacter — no boundary crossed. |
| elixir.bat.md:57 / THREAT_MODEL:38 | `elixir.bat` ↔ `elixir` (sh) flag-by-flag / shift-count parity | **Unresolved** | elixir.bat.md:57 explicitly did NOT audit shift-count parity; no other report performs the flag-by-flag comparison. Coverage gap, not a defect — ledger row only. |

## Findings

#### `CSCAN-crossfile-elixir-cdpath-hijack-untracked`
- **Finding ID** — `CSCAN-crossfile-elixir-cdpath-hijack-untracked`
- **Severity** — Medium
- **Confidence** — high
- **Files involved** — iex.md, elixirc.md, elixir.md
- **Issue** — Both `iex.md` (finding `CSCAN-iex-cdpath-hijack`, cross-file note at iex.md:34/48) and `elixirc.md` (elixirc.md:56) discharge the CDPATH-hijack defect in the sibling `elixir` script by *deferring it*: iex.md states the byte-identical `readlink_f` (`elixir:69-77`, `diff` reported IDENTICAL) "has the identical `CDPATH` exposure and needs the same fix… it is a defect of `elixir`, **tracked there**, not double-counted against `iex`." But `elixir.md` does not track it. The elixir report contains no CDPATH finding, and its only adjacent ruled-out entry (elixir.md:58) dismisses the `SELF`/`SCRIPT_PATH` → `../lib/elixir/ebin` concern as "already-privileged; not a lower-trust boundary in documented invocation." That dismissal is exactly what iex.md's *verified* evidence contradicts: `CDPATH` is an inherited environment variable (a lower-trust input — CI vars, direnv, Docker `ENV`) that silently poisons `SCRIPT_PATH` via `cd "$(dirname "$1")"` without the attacker controlling the script's location or any symlink. The verified `iex` exploit runs unchanged against `elixir`'s byte-identical `readlink_f` and its absolute-path `exec` at `elixir:219`/`245`. So a Medium-severity, verified-mechanism code-execution defect in `elixir` is asserted-away by two callers as "tracked elsewhere" while its own report rules the closest concern out — nobody actually owns it.
- **Resolution** — Upgrade the `elixir` report: mint a CDPATH-hijack finding against `elixir:69-77`/`elixir:219`/`elixir:245` (same class as `CSCAN-iex-cdpath-hijack`), and correct elixir.md:58's "already-privileged" dismissal, which does not account for `CDPATH` poisoning of the `cd` that computes `SCRIPT_PATH`. Fix (`unset CDPATH` / `CDPATH= cd …`) must land in `elixir`, not only `iex`.

#### `CSCAN-crossfile-mix-readlink-parity-overclaim`
- **Finding ID** — `CSCAN-crossfile-mix-readlink-parity-overclaim`
- **Severity** — Low
- **Confidence** — high
- **Files involved** — elixirc.md, mix.md
- **Issue** — elixirc.md:56 states the `readlink_f`/CDPATH "fix should be applied consistently across `elixir`, `iex`, and **`mix`** to avoid a fixed-here-but-not-there gap," while hedging that "`iex`/`mix` were not read in this pass… parity there is stated, not verified." mix.md:28 shows `mix` is not a POSIX shell script at all — it is an Elixir script (`#!/usr/bin/env elixir`) whose entire body is `Mix.CLI.main()`, containing no `readlink_f`, no `cd`, and no CDPATH-sensitive path resolution (verified by grep in mix.md). The parity claim is therefore contradicted: `mix` has no readlink_f defect to fix. This is an over-claim (remediation misdirected at a file that lacks the sink), not a hidden vulnerability, so severity is Low — but a downstream reader acting on elixirc.md would waste effort hunting a non-existent defect in `mix` and might miss that the real parity set is `{elixir, iex, elixirc}` plus the Windows `.bat` chain.
- **Resolution** — Correct elixirc.md's parity statement to scope the CDPATH fix to the three sh scripts that actually carry `readlink_f` (`elixir`, `iex`, `elixirc`); drop `mix` from that set. No source change needed in `mix`.
