# /Users/lukas/code/claude-scanner/elixir/bin/elixirc.bat
<!-- sha256: 1a3d323f73598f1ef0476409c59c8dff899970704660ac97f30381aa80be57a5 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/bin/elixirc.bat
<!-- sha256: 1a3d323f73598f1ef0476409c59c8dff899970704660ac97f30381aa80be57a5 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-27_

## 1. Summary
`elixirc.bat` is a thin Windows launcher: it runs a help/argument-counting pre-scan (`for %%A in (%*)`, lines 9-15) and then forwards every argument verbatim to `elixir.bat` via `call "%~dp0\elixir.bat" +elixirc %*` (line 39). This pass re-examined the first-pass command-injection finding against documented cmd.exe FOR-variable semantics and downgraded it (see §2 and "Dropped/downgraded"), and grounded every cross-file hedge by reading `elixir.bat` and the sh sibling `elixirc` directly.

## 2. Security

#### `CSCAN-elixirc-for-loop-prescan-parse` (downgraded from `CSCAN-elixirc-for-loop-quote-injection`)
- **Finding ID:** `CSCAN-elixirc-for-loop-prescan-parse`
- **Severity:** Low
- **Confidence:** low
- **Location:** `elixirc.bat:9-15` — the `for %%A in (%*)` pre-scan and its `if /I "%%A"=="…"` comparisons (lines 10-13).
- **Class:** Windows batch argument parsing / quote-balance robustness (candidate command-injection, unconfirmed).
- **Impact (unconfirmed):** A quote-bearing argument (e.g. a lone `"` token, or `"&"`-style interior quoting) substituted textually into `if /I "%%A"=="--help"` can unbalance the quotes on the comparison line and mis-parse it. Whether this rises to *command execution* is unconfirmed and, on the documented semantics below, unlikely.
- **Why this is downgraded, not upgraded:** The first pass asserted a clean `" & calc & "` → `calc` breakout inside the counting loop. Re-examining the mechanism against documented cmd.exe behaviour undermines that specific claim on two independent points:
  1. **FOR-set tokenisation.** `for %%A` (no `/F`) splits its set on whitespace/`,`/`;`/tab and does *not* honour quotes as grouping. So `%*` = `" & calc & "` does **not** yield a single token `" & calc & "`; it yields several tokens (`"`, `&`, `calc`, `&`, `"`). The first-pass "single token substituted whole" mechanism is therefore inaccurate.
  2. **FOR-variable late expansion.** A plain `%%A` reference is expanded in a late parse phase whose result is *not* re-scanned for command operators (`&`, `|`). The widely-documented consequence is that `for %%a in ("a & b") do echo %%a` prints the string literally and does **not** run a second command. That behaviour argues directly *against* the `&`-injection interpretation.\
  The residual concern that survives is narrower and weaker: a token that is (or contains) an *unbalanced quote* can still perturb the `if /I "…"=="…"` comparison's quote parity, potentially mis-firing `goto documentation` or miscounting `argc`. That is a robustness/parse-correctness bug at most; I have no executed evidence it reaches command execution.
- **Evidence:** `asserted` — unexecuted. The sandbox has no `cmd.exe`/`wine` (only `erl`, which cannot parse batch), so neither the original breakout nor the residual quote-imbalance behaviour can be reproduced here. Confidence is therefore low, and the finding is retained (not dropped) only as a pointer for confirmation on a real Windows shell with the harmless probe `elixirc.bat " & echo INJECTED & "` and `elixirc.bat a"b`.
- **Suggested fix (defensive, cheap):** Drop the `for %%A in (%*)` pre-scan and iterate with `shift`, comparing the de-quoted `"%~1"`; or mirror the delayed-expansion quote-escaping already used in `elixir.bat:94-100` (`set "par=%~1"`, `set par="!par:"=\"!"`, compare `!par!`). `%%~A` alone strips only the outer quote pair and is insufficient.

### Ruled out
- **`call "%~dp0\elixir.bat" +elixirc %*` (line 39) — quote breakout of the forwarded tail.**
  - Axes examined: prefix quote-parity (the fixed `"%~dp0\elixir.bat"` prefix contributes an even number of `"`, since Windows paths cannot contain `"`); faithful-forwarding parity against the sh sibling (`elixirc:43` = `exec "$SCRIPT_PATH"/elixir +elixirc "$@"` — I read it this pass and confirmed identical intent: forward all args, prepend the `+elixirc` selector).
  - Not examined: runtime cmd.exe re-parse of `%*` at the `call` site (needs Windows to execute). This axis is unexecuted, so the entry stays `asserted`; reading `elixir.bat`/`elixirc` cannot clear it.
- **PATH / CWD hijack of the delegate.** Not present. `elixir.bat` is invoked via the fully-qualified quoted `"%~dp0\elixir.bat"`, not by bare name, so cmd's current-directory-first batch search cannot load a planted `elixir.bat`. (Verified by reading line 39; no execution needed for this static property.)
- **Argument injection of a leading-`-`/`+` filename into Elixir/Erlang switches — sink is not in this file.** Confirmed by reading the delegate: the switch-routing parser is `elixir.bat:93-132` (`:startloop`), which is where a `-`/`+`-leading value would be interpreted (e.g. the `parsErlang` assignments at lines 121-131 pass values into real `erl` switches like `-boot`, `-config`, `-setcookie`, `-args_file`). `elixirc.bat` only prepends the literal `+elixirc` selector (consumed harmlessly at `elixir.bat:103`) and forwards `%*` unchanged. The sh sibling has identical structure (`elixirc:43`). Nothing `elixirc.bat`-specific to fix; the finding, if any, belongs to `elixir.bat`.
- **`echo Usage: %~nx0` (line 20) — `&` in the script's own filename.** Requires renaming the installed `elixirc.bat` to contain `&`; that is the installer's privilege (same trust level as the script), not an external input crossing a boundary. Not a finding.
- **Unquoted `%*` globbing in the FOR set (line 9).** A `*`/`?` argument is filename-expanded during counting, perturbing `argc`/help detection, but the loop variable feeds no executed command — line 39 forwards the raw `%*`, not the expanded tokens. Functional quirk, no security impact.

### Cross-file caveat (now read, still not this file's defect)
The richer injection surface is in the delegate, and I read it this pass to confirm it exists rather than assert it: `elixir.bat:143` / `:145` build the final command line as `"%ERTS_BIN%erl.exe" %ELIXIR_ERL_OPTIONS% %parsErlang% %beforeExtra% -extra %*` with `%ELIXIR_ERL_OPTIONS%`, `%parsErlang%`, and `%*` all unquoted after leaving `setlocal disabledelayedexpansion` (line 135). Those unquoted expansions are the interesting sink and are `elixir.bat`'s to answer for; `elixirc.bat` contributes only faithfully-forwarded `%*`. Flagged for the `elixir.bat` review, not fixed here.

### Minor parity note (non-security)
The sh `elixirc` triggers help only on `$# -eq 0`, `$1 = --help`, or `$1 = -h` (checks the *first* arg). `elixirc.bat` scans *every* arg (lines 9-15) and additionally treats `/h` and `/?` as help. This is a behavioural divergence between the two hand-written front-ends but has no security consequence; noted for completeness after reading both.

## 3. Risks / caveats
- **No live reproduction.** Linux sandbox, no `cmd.exe`/`wine`; `erl` is present but irrelevant to batch parsing. Every batch-parse claim above is reasoned, not executed. The one axis that would need execution — cmd.exe's runtime re-parse of `%%A` and of `%*` at the `call` — remains on the `Not examined` list for the relevant ruled-out entry and caps §2 at low confidence.
- **Reading neighbours ≠ executing.** This pass read `elixir.bat` and `elixirc` (sh) to ground the cross-file hedges; that confirms *where the sinks live* and *that forwarding is faithful*, but it does not and cannot substitute for running the batch parser, so no ruled-out axis was cleared on the strength of a read alone.

## 4. Did I verify this?
- Read `elixirc.bat` (target), `elixirc` (sh sibling), and `elixir.bat` (delegate) in full this pass. ✅ (static reads)
- Executed the FOR-loop pre-scan or the `call` forwarding under a real cmd.exe. ❌ (no cmd.exe/wine in sandbox)
- Confirmed the sink for `-`/`+`-leading argument interpretation is `elixir.bat:93-132`, not this file. ✅ (by reading)
- Confirmed faithful `%*` forwarding parity with sh `elixirc:43`. ✅ (by reading)
- Confirmed the FOR-variable command-injection mechanism by execution. ❌ — documented cmd.exe FOR semantics argue against it; retained only as a low-confidence, Windows-confirm-needed parse-robustness finding.

## Dropped / downgraded in re-review
- **`CSCAN-elixirc-for-loop-quote-injection` (Medium, "asserted") → downgraded to `CSCAN-elixirc-for-loop-prescan-parse` (Low).** Not dropped outright — a residual quote-balance robustness concern in the pre-scan survives — but the first-pass *command-injection* framing and its `" & calc & "` → `calc` proof-of-concept are withdrawn. New basis: (1) non-`/F` FOR splits its set on whitespace and does not preserve `" & calc & "` as one token, so the stated PoC token stream is wrong; (2) plain `%%A` FOR-variable references are expanded in a late phase not re-scanned for `&`/`|` operators (the documented `for %%a in ("a & b") do echo %%a` prints literally), which contradicts the asserted breakout. Because I cannot execute batch here, I downgrade rather than confirm-either-way, per "prefer downgrading to upgrading."
- No first-pass finding was silently removed.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 2 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Argument injection of a leading-`-`/`+` filename into Elixir switches:** Real for the toolchain but not a defect *of this file*. `elixirc.bat` forwards `%*` verbatim to `elixir.bat`, whose `:startloop` parser (lines 102-132) is the sink; the behavior is identical in the sh `elixirc` (`exec … +elixirc "$@"`), so parity is preserved and there is nothing `elixirc.bat`-specific to fix here.

- **`echo Usage: %~nx0` (line 20) — `&` in script filename:** Requires the `elixirc.bat` file itself to be renamed to contain `&`; that is chosen by whoever installs the launcher (same trust level as the script), not an external input crossing a boundary. Not a finding.
- **Unquoted `%*` globbing in the `for` set (line 9):** A `*`/`?` argument is filename-expanded during counting, perturbing `argc` and help detection, but the loop variable never feeds the executed command (line 39 uses `%*` directly). Functional quirk, no security impact.
