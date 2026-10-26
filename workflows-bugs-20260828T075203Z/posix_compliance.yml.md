# /Users/lukas/code/claude-scanner/elixir/.github/workflows/posix_compliance.yml
<!-- sha256: 057c5447d2b636c1b7ac44263edb83f6788de06e0acb6878162acbe8036ad674 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T07:59:37Z_

# POSIX Compliance Workflow — Bugfix Review

## Summary
The workflow's sole ShellCheck step swallows violations in `bin/elixir` and `bin/elixirc` because `shellcheck … && echo` places `shellcheck` in the non-final position of an `&&` list, which `set -e` exempts from failing the job.

## Bugfixes

#### `CSCAN-posix_compliance-shellcheck-failure-swallowed`
- **Finding ID:** `CSCAN-posix_compliance-shellcheck-failure-swallowed`
- **Severity:** High
- **Confidence:** high
- **Location:** `posix_compliance.yml:44-53`, job `check_posix_compliance`, step "Run ShellCheck on bin/ dir"
- **Class:** Steps that cannot fail the build
- **Issue:** GitHub Actions runs `run:` blocks under `bash --noprofile --norc -eo pipefail`. Under `set -e` (errexit), POSIX/bash exempt from exit "any command of an AND-OR list other than the last." The step is written as three lines of the form `shellcheck <file> && echo "<file> is POSIX compliant"`. In each line `shellcheck` is the *first*, not the *last*, command in the `&&` list, so when ShellCheck reports violations (exit ≠ 0) the failure is exempt from errexit, the `echo` is skipped, and execution simply falls through to the next line. Only the final `shellcheck bin/iex` invocation can actually fail the job — because its non-zero status becomes the script's last exit status. Violations in `bin/elixir` or `bin/elixirc` never fail the job.
- **Impact:** The workflow exists to enforce POSIX/ShellCheck compliance on the three launcher scripts, yet non-compliance introduced into `bin/elixir` or `bin/elixirc` (two of the three targets, including the one with special excludes) passes CI green. A `pull_request` that regresses POSIX-compliance in those files sails through the check unnoticed; the class of defect that escapes is exactly the one this file was created to catch. The green check gives false assurance to maintainers and reviewers.
- **Evidence:** `verified`. Minimal errexit reproduction: `bash --noprofile --norc -eo pipefail -c 'false && echo x; echo REACHED'` prints `REACHED` and exits `0`. Full-step reproduction with a stub `shellcheck` that fails only on `bin/elixir` (the first, non-last invocation):
  ```
  shellcheck -e SC2039,2086 bin/elixir && echo "bin/elixir is POSIX compliant"
  shellcheck bin/elixirc && echo "bin/elixirc is POSIX compliant"
  shellcheck bin/iex && echo "bin/iex is POSIX compliant"
  ```
  Output:
  ```
  SHELLCHECK: violation in bin/elixir
  bin/elixirc is POSIX compliant
  bin/iex is POSIX compliant
  STEP EXIT CODE: 0
  ```
  The `bin/elixir` violation is reported to stderr but the step exits `0`.
- **Suggested fix:** Drop the `&& echo` gating so each ShellCheck failure propagates under errexit, e.g.:
  ```yaml
  run: |
    shellcheck -e SC2039,2086 bin/elixir
    shellcheck bin/elixirc
    shellcheck bin/iex
  ```
  (Optionally re-add the `echo` lines as separate unconditional statements.) Alternatively, invert to `shellcheck … || { echo "…not compliant"; exit 1; }`.

## Risks / caveats
- I could not run real `shellcheck` (not installed in this environment), so the reproduction used a stub returning the same exit codes; the errexit/AND-OR behaviour that drives the finding is the runner's shell semantics, which I verified directly, not ShellCheck's internals.
- I confirmed GitHub's default Linux shell is bash-with-`-eo pipefail`; the AND-OR errexit exemption is POSIX-standard, so the bug holds equally if the runner falls back to `sh -e`.

### Ruled out
- **`paths:` / trigger filters miss the checked files** — Ruled out by reading lines 7-18: the anchored `paths-filter` lists exactly `bin/elixir`, `bin/elixirc`, `bin/iex` plus the workflow file, and is applied to both `push` and `pull_request`; `workflow_dispatch` allows manual runs regardless. The filter includes precisely the files the step inspects.
- **`fail-fast: false` masking a leg** — Ruled out: there is no `matrix`, so `fail-fast` governs nothing; it neither adds nor removes coverage.
- **Mixed exclude-code format `-e SC2039,2086` (bare `2086` without `SC`)** — Not raised as a finding: ShellCheck accepts both `SCxxxx` and bare-numeric codes in `--exclude`, so `2086` is valid; I could not execute ShellCheck here to re-verify, but even if it were rejected the effect would be a hard error (louder, not silent), which is not the silent-CI failure class under review.
- **Unpinned/floating action** — Ruled out: the only `uses:` (`actions/checkout`) is pinned to a full commit SHA (line 35).
- **Over-broad permissions** — Ruled out: `permissions: contents: read` (lines 20-21) is least-privilege and `persist-credentials: false` is set on checkout (line 37).

