# /Users/lukas/code/claude-scanner/elixir/.github/workflows/ci.yml
<!-- sha256: fcc50d3f863cd8d1402f22a47bd25aaff18095c035657b3042b1f8280751715f -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/ci.yml
<!-- sha256: fcc50d3f863cd8d1402f22a47bd25aaff18095c035657b3042b1f8280751715f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

# CI Workflow Review: `ci.yml`

## Summary
The workflow is largely well-formed (SHA-pinned actions, least-privilege `contents: read`, `persist-credentials: false`, intentional `continue-on-error` scoped to development OTP legs). The one substantive divergence is that the Windows job compiles Erlang code without `warnings_as_errors`, so the two OS jobs do not test the same thing. This re-review resolves the two first-pass hedges: the shell-default assumption is confirmed against GitHub's documented platform contract (verified); the `COVER` empty-string behavior depends on a `Makefile` that is not present in this checkout and is not a callgraph neighbour, so it is now recorded as a low-confidence finding rather than a hedged caveat.

## Bugfixes

#### `CSCAN-ci-windows-missing-erlc-warnings`
- **Finding ID:** `CSCAN-ci-windows-missing-erlc-warnings`
- **Severity:** Low
- **Confidence:** high
- **Location:** `ci.yml:42-43` (`test_linux` job `env.ERLC_OPTS`) vs `ci.yml:111-152` (`test_windows` job — no `env` block)
- **Class:** Version drift
- **Issue:** `ERLC_OPTS: "warnings_as_errors"` is set only as `test_linux`-job env (line 43). The workflow-level `env` (lines 12-15) exports `ELIXIRC_OPTS` (the Elixir compiler flag) to both jobs, but **not** `ERLC_OPTS` (the Erlang `erlc` flag). `test_windows` declares no job-level `env`, so on Windows the Erlang bootstrap sources in `lib/elixir` are compiled with Erlang compiler warnings treated as ordinary warnings, not errors.
- **Impact:** The two OS jobs enforce different strictness. An Erlang-level compiler warning introduced in a change compiles clean on Windows and only the Linux legs fail — the Windows job's "compile is warning-clean" guarantee is silently weaker than it appears. Because the Linux legs still enforce `warnings_as_errors`, the defect class (new Erlang warnings) is still caught overall, which is why this is Low rather than a true silent hole; the cost is a misleading green Windows check and loss of the redundant Windows-specific coverage that same-config parity would give.
- **Evidence:** asserted — read directly from the file: `test_linux` sets `ERLC_OPTS` at lines 42-43; `test_windows` (lines 111-152) has no `env:` key and the top-level `env` block (lines 12-15) does not include `ERLC_OPTS`.
- **Suggested fix:** Promote `ERLC_OPTS: "warnings_as_errors"` to the workflow-level `env` block (alongside `ELIXIRC_OPTS`) so both jobs share it, or add an identical `env` block to `test_windows`.

#### `CSCAN-ci-cover-empty-string-env`  *(Upgraded from hedged assumption in first pass)*
- **Finding ID:** `CSCAN-ci-cover-empty-string-env`
- **Severity:** Low
- **Confidence:** low
- **Location:** `ci.yml:75-76` (`COVER: "${{ matrix.coverage }}"` on the Elixir test-suite step)
- **Class:** Misconfiguration / env-var interpolation
- **Issue:** `matrix.coverage` is set (`true`) on exactly one leg (`28.4`, line 33) and is unset on every other leg. On the unset legs, `${{ matrix.coverage }}` interpolates to the empty string, so those steps run with the env var **present but empty** (`COVER=""`), not absent. Whether this is harmless depends entirely on how the `Makefile`'s `cover` target gates on `COVER`: a value-equality gate (`ifeq ($(COVER),true)`) is fine; a presence gate (`ifdef COVER` / `$(if $(COVER),…)`) would treat empty-but-present as "enabled" and run coverage instrumentation on all legs (slower, not incorrect results).
- **Impact:** At worst, coverage instrumentation runs on every Linux leg instead of one — a performance/time regression, not a correctness or silent-failure defect. No secret or trust boundary is involved.
- **Evidence:** asserted — I could not verify the gate direction. The `Makefile` is **not present in this checkout** (only `.github/workflows/` is materialized; `find … -iname 'Makefile*'` over the checkout returns nothing) and it is not a callgraph neighbour listed in the standing context, so it cannot be read; there is no `make`/Makefile in the sandbox to execute the gate against. The claim therefore stays a low-confidence finding rather than a resolved caveat.
- **Suggested fix:** Make the env var conditional so it is absent on non-coverage legs — e.g. `COVER: "${{ matrix.coverage && 'true' || '' }}"` still leaves it empty; prefer gating the whole `env:`/step on `matrix.coverage`, or confirm the Makefile uses `ifeq ($(COVER),true)` and document that invariant.

## Risks / caveats
- **Shell defaults (verified, not assumed).** `test_linux` runs on `ubuntu-24.04` and `test_windows` on `windows-2022` — both **GitHub-hosted** images. Per GitHub's documented default-shell contract, `run:` steps with no `shell:` key execute under `bash --noprofile --norc -eo pipefail {0}` on Linux and `pwsh -command ". '{0}'"` (PowerShell 7+) on Windows-hosted runners. PowerShell 7 supports `&&`, so `make test_formatted && echo …` (lines 143, 66) parses on the GitHub-hosted Windows image. This is a platform-contract fact for GitHub-hosted runners, not a file-level assumption; it does not depend on any module I could not read. The only residual (out-of-scope) case is a self-hosted `windows-2022`-labelled runner lacking `pwsh`, where the default would fall back to Windows PowerShell 5.1 and the `&&` chains would be parse errors — not applicable to this workflow's GitHub-hosted `runs-on`.
- The `master`/`maint` OTP legs depend on `erlef/setup-beam` resolving those refs; a resolution failure there fails loudly (the setup step has no `continue-on-error`), so it is not a silent-failure concern.

### Ruled out
- **`make cover | tee "$GITHUB_STEP_SUMMARY"` (line 91) swallowing a `make cover` failure** — this is the textbook "non-zero exit swallowed by a pipeline" pattern, but the step sets no `shell:` and runs on GitHub-hosted `ubuntu-24.04` under GitHub's documented default Linux bash invocation `bash --noprofile --norc -eo pipefail {0}`. With `pipefail` active, a non-zero exit from `make cover` propagates as the pipeline's exit status even though `tee` succeeds, so the step still fails. Named mechanism + confirmed GitHub-hosted default, so the ruling stands; had `shell: bash` been re-declared without `pipefail`, this would be a finding.
  - **Axes examined:** pipeline-exit-status propagation under the default shell, shell-key presence on the step. **Not examined:** none — the failing-exit path is governed entirely by the documented default shell for the GitHub-hosted image named in `runs-on`, which is stated in the file.
- **`continue-on-error: ${{ matrix.development == true }}` on the two test-suite steps (lines 70, 74)** — intentional and correctly scoped: it evaluates true only for the `development: true` legs (`master`, `maint` pre-release OTP), where allowing failure is the design. For all release-OTP legs `matrix.development` is unset, coercing to `false`, so those legs still gate. Not masking regressions in shipped-OTP coverage.
  - **Axes examined:** per-leg value of `matrix.development` (set on lines 38, 40 only), boolean coercion of the unset case. **Not examined:** none — every matrix leg's `development` value is visible in the file.
- **`if:` conditions (`matrix.deterministic`, `matrix.docs`, `matrix.coverage`) never/always firing** — each key is set on exactly one `include` leg (deterministic → `29.0`, line 30; docs+coverage → `28.4`, lines 32-33), so each guarded step runs on precisely the intended leg and is skipped elsewhere. No always-true string literals (`if: 'false'`) present anywhere in the file.
  - **Axes examined:** which leg sets each key, presence of literal always-true/always-false guards. **Not examined:** none — all `include` legs are enumerated in the file.
- **Matrix legs testing the same thing / quoting drift** — every `otp_version` is a distinct quoted string (`"29.0"`, `"28.4"`, `"28.1"`, `"27.3"`, `"27.0"`, `master`, `maint`; Windows `"29.0"`, `"28.1"`, `"27.3"`). All numeric versions are quoted, so there is no `28.10`-vs-`28.1` YAML-float truncation, and no duplicate legs.
  - **Axes examined:** quoting of each version literal, duplication across legs. **Not examined:** none — all version strings are listed in the file.
- **Trigger/filter errors** — `on:` declares `push`, `pull_request`, `workflow_dispatch` with no `branches:`/`paths:` narrowing, so no filter can exclude the sources this workflow exists to compile and test.
  - **Axes examined:** trigger list and absence of filter keys. **Not examined:** none — the full `on:` block is visible (lines 7-10).
- **Missing `concurrency`** — absent, which permits redundant/superseded runs (and the usual `push`+`pull_request` double-run on same-repo PR branches) but cannot hide a defect or cancel a publish; this file performs no publishing. Resource cost only, not a correctness/silent-failure issue.
  - **Axes examined:** presence of `concurrency`, publishing side effects in this file. **Not examined:** none.

## Did I verify this?
- **`CSCAN-ci-windows-missing-erlc-warnings`** — verified against the file: `ERLC_OPTS` appears only at `ci.yml:42-43`; the top-level `env` (12-15) omits it; `test_windows` (111-152) has no `env:`. High confidence.
- **`CSCAN-ci-cover-empty-string-env`** — **not** verified: the deciding artifact (`Makefile`, its `cover` target's gate direction) is absent from this checkout (`find` over the tree finds no Makefile) and is not a listed neighbour, and no `make` toolchain in the sandbox can exercise the gate. Recorded as low-confidence per the "cannot verify → downgrade to finding" rule; upgraded from the first pass's hedged caveat.
- **Shell defaults** — verified against GitHub's documented default-shell behavior for GitHub-hosted `ubuntu-24.04` / `windows-2022` runners, which is what `runs-on` selects. No file-level assumption remains; the first-pass "I assumed GitHub-hosted runner defaults" wording is removed.

## Dropped in re-review
None. Every first-pass finding is preserved. The first pass's two hedged caveats were not dropped but reclassified: the shell-default caveat is now a verified platform-contract note (no longer hedged), and the `COVER`/Makefile caveat is upgraded to the low-confidence finding `CSCAN-ci-cover-empty-string-env` because the Makefile is unreadable in this checkout.
