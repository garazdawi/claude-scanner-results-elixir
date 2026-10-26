# Cross-file findings

No per-file report uses the machine-extractable `* Depends on:` line format; the reviewers instead expressed cross-file dependencies as prose hedges ("verified by reading the neighbour", "out of scope, lives in `elixir`", "asserted from cmd.exe semantics"). The ledger below enumerates every such asserted cross-file dependency and resolves it against the cited report.

## Asserted-dependency ledger

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| `mix.bat.md` | `elixir.bat` parser: first positional `"%~dp0\mix"` matches no arm → falls to `:run`, user args reach Mix via `-extra %*` | Confirmed | `elixir.bat.md` "Ruled out" + `mix.md` both confirm a non-switch script path hits the `*)`/`:run` catch-all. |
| `mix.md` | `elixir`/`elixir.bat` hold all launcher machinery; the `mix` path falls into `elixir`'s `*)` catch-all (`elixir:190-197`) | Confirmed | `mix.md` read `elixir` 1-247 and `mix.bat`; `elixir.md` ruled-out entry confirms catch-all copies remaining argv. |
| `mix.md` | `Mix.CLI.main()` correctness (command dispatch) | Unresolved | Module not in repo (Glob `**/mix/cli.ex` empty); nobody adjudicated Mix dispatch — coverage gap, not a defect. |
| `iex.md` | `elixir` handles `--no-halt` C=1 (`elixir:120-122`) | Confirmed | `iex.md` verified by reading `elixir`; `elixir.md` ruled-out cross-checked all C/S arities to USAGE. |
| `iex.md` | `elixir` handles `--erl` S=2, word-split via unquoted `$ERL` (`elixir:145-148,229`) | Confirmed | `iex.md` verified; `elixir.md` notes line 229 unquoting is intended array reconstruction. |
| `iex.md` | `elixir` `+iex` C=1 is the sole arm setting `MODE=iex` (`elixir:116-119,222`) | Confirmed | `iex.md` verified ordering invariant by reading `elixir`. |
| `iex.md` | `elixir` handles IEx switches `--dbg`/`--dot-iex`/`--remsh` (`elixir:123`, C=2) | Confirmed | `iex.md` verified against `elixir` parser. |
| `iex.bat.md` | `elixir.bat` handles `--no-halt`(114), `--erl`(131), `+iex`(102), `--remsh`/`--dot-iex`/`--dbg`(117-119); `useIEx` gate (134-137) | Confirmed | `iex.bat.md` read `elixir.bat` in full and cited each arm. |
| `iex.bat.md` | cmd.exe tokenisation of `""%1""`, `%*`-vs-`shift`, `\\` collapse | Unresolved | No `cmd.exe` in sandbox; entries stay `asserted` — coverage gap, self-declared. |
| `elixirc.md` | `elixir` `+elixirc` C=1 (`elixir:113-115`) + catch-all forwarding (`elixir:190-197`) | Confirmed | `elixirc.md` verified by read + `ELIXIR_CLI_DRY_RUN=1` execution. |
| `elixirc.bat.md` | `elixir.bat` correctly handles `+elixirc` and `-extra %*` arities | Unresolved | Explicitly "not adjudicated here"; no report verifies a `+elixirc` arm in `elixir.bat` (`iex.bat.md` enumerated arms but not this one). |
| `elixir.bat.md` | `--pipe-to` rejection exit code / stdout-vs-stderr | Unresolved | `asserted` from cmd semantics; no Windows host — control-flow reasoning only. |
| `elixir.bat.md` / `mix.bat.md` | `%*` re-passes distribution opts (`--cookie`/`--name`/`--boot`) after `-extra`; Elixir CLI tolerates the duplication | Unresolved | Rests on unverified Mix.CLI/Elixir-CLI-parser behaviour, which is out of repo — inherited risk, unchecked. |
| `mix.ps1.md` | `mix.bat` sits beside the script; `& $mixBatPath` launches it; failure surface depends on caller `$ErrorActionPreference` | Unresolved | No PowerShell interpreter; finding self-contained but the `$null`-vs-terminating-error surface is unchecked. |

## Findings

#### `CSCAN-crossfile-windows-launcher-error-exit-zero`

- **Finding ID:** `CSCAN-crossfile-windows-launcher-error-exit-zero`
- **Severity:** Low
- **Confidence:** medium
- **Files involved:** `elixir.bat.md`, `elixir.md`, `iex.bat.md`, `iex.md`, `elixirc.bat.md`, `elixirc.md`, `THREAT_MODEL.md`, `RFC_OBLIGATIONS.md`
- **Issue:** `THREAT_MODEL.md` (line 31) and `RFC_OBLIGATIONS.md` claim 9 (WINDOWS_PARITY) require the `.bat` variants to be behaviourally equivalent to their `sh` siblings. The per-file reports, read together, show a **systemic divergence in exit status on non-run paths**: every POSIX launcher terminates non-boot/help paths with `exit 1`, while every Windows sibling reaches `:end`→`endlocal`→EOF after a successful `echo`, exiting `0`. Concretely — POSIX: `elixir:185,187` (`--pipe-to` switch-rejection `exit 1`), `iex:24` (usage → stderr, `exit 1`), `elixirc:9` (help → `exit 1`); Windows: `elixir.bat:132`→`:end`(148) (`--pipe-to` unsupported message, exit 0 — already minted as `CSCAN-elixir-pipe-to-rejection-exit-zero`), `iex.bat:14-26` (usage → stdout, exit 0), `elixirc.bat` documentation path (help, exit 0). Individually the per-file reviewers dismissed the help-path cases as "platform convention / cosmetic," but as a single cross-file pattern it means any CI wrapper, `mix` task, or batch caller that branches on `%ERRORLEVEL%` treats a refused/aborted launch on Windows as success — the same failure mode the `--pipe-to` finding already confirmed, generalised across the suite.
- **Resolution:** Decide whether WINDOWS_PARITY governs exit status (not just switch-set coverage). If yes, the help/usage `.bat` paths should `exit /B 1` (and write to stderr) to match the `sh` siblings, promoting the currently-"cosmetic" `iex.bat`/`elixirc.bat` divergences to defects alongside the already-filed `--pipe-to` one. Confirm on a real Windows host (no reviewer could execute cmd.exe).

The remaining ledger rows are **Unresolved coverage gaps, not defects**: `+elixirc` handling in `elixir.bat`, Mix.CLI dispatch, the duplicated distribution-option pass-through, and the cmd.exe/PowerShell tokenisation axes are all unverified because the originating implementation is out of repo or no interpreter was available. Per the enumeration rules they remain ledger rows and are not minted as findings — resolving them requires reading `elixir.bat`'s parse loop for a `+elixirc` arm and the compiled `Mix.CLI` source, neither adjudicated in this report set.
