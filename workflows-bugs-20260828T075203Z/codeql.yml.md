# /Users/lukas/code/claude-scanner/elixir/.github/workflows/codeql.yml
<!-- sha256: 61af0bab406c6e01efd732d9e2be3d9f876dfabf3bc09457b42775d9764e0347 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/codeql.yml
<!-- sha256: 61af0bab406c6e01efd732d9e2be3d9f876dfabf3bc09457b42775d9764e0347 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

# CodeQL Advanced workflow review — `codeql.yml`

## Summary
The workflow is small and well-formed. Top-level permissions are least-privilege (`contents: read`), each job scopes up only to `security-events: write` for SARIF upload, both checkouts set `persist-credentials: false`, and every action is SHA-pinned. One low-confidence finding remains: the `zizmor` gate's blocking behavior depends on the pinned action's default exit semantics, which cannot be verified in this environment (the action lives on GitHub behind a SHA, not as a local file, and network fetch is unavailable here). That finding is retained rather than upgraded or dropped, because "does the job turn red on findings" is genuinely undetermined from the file alone.

## Bugfixes

#### `CSCAN-codeql-zizmor-gate-may-not-fail`
- **Finding ID** — `CSCAN-codeql-zizmor-gate-may-not-fail`
- **Severity** — Low
- **Confidence** — low
- **Location** — `codeql.yml:54-55`, job `zizmor`, step "Run zizmor" (`zizmorcore/zizmor-action@3dc1ecc…` v0.6.2)
- **Class** — Steps that cannot fail the build
- **Issue** — The `zizmor` job exists to gate on workflow-security findings, but whether a finding turns the job red is controlled entirely by the pinned action's default exit behavior. All tuning inputs (`min-severity`, `min-confidence`, `persona`) are left unset (the step has no `with:` block at all, lines 54-55). If the action's default is to upload SARIF and exit `0`, the job is green even when zizmor reports issues, and the gate exists only as a Security-tab alert, not a PR blocker.
- **Impact** — If the default is non-failing, every zizmor-detectable defect class (script injection, over-broad permissions, unpinned actions in *other* workflows) can merge through a green check. The check appears to run but never blocks.
- **Evidence** — `asserted`. The action's `action.yml` for the pinned SHA `3dc1ecc…` could not be read: it is not a file in this repository (it is fetched by GitHub at runtime from `zizmorcore/zizmor-action`), and `WebFetch`/network access is not available in this non-interactive session. zizmor's CLI exits non-zero on findings, and the action is generally understood to propagate that by default — which is why this is rated **low** confidence and most likely *not* a bug, rather than a confirmed finding. It stays a finding (not a dismissal) precisely because the deciding fact is unverifiable here.
- **Suggested fix** — If a blocking gate is intended, set an explicit `min-severity`/`min-confidence` and confirm the action's failure mode against the pinned SHA; if only alerting into the Security tab is intended, no change is needed.

## Risks / caveats
- I reviewed only `codeql.yml`. The two pinned third-party actions (`github/codeql-action`, `zizmorcore/zizmor-action`) are not local files and could not be fetched, so their default failure semantics and permission requirements are asserted from prior knowledge, not verified against the pinned SHAs. The only claim that actually turns on this — the zizmor exit behavior — is carried above as a low-confidence finding rather than a silent assumption.
- Both jobs' SARIF uploads rely on `security-events: write`, which GitHub silently downgrades to read-only for `pull_request` runs from forks. This is standard platform behavior, not a file defect, but it means fork-PR runs will not populate alerts. Not raised as a finding.

## Ruled out

- **Trigger/paths filters excluding target files** — `Not examined: none.` The `on:` block (lines 6-12) is complete: `push`/`pull_request` on `main` plus a weekly `schedule` cron, with **no** `paths:`/`paths-ignore:` filter. CodeQL fires on every change to `main`, so it cannot be silently narrowed away. Verified by reading the full trigger block.
- **`if:` conditions that never/always fire** — `Not examined: none.` There are no `if:` clauses anywhere in the 56-line file. No always-true/always-false condition can exist. Verified by reading every line.
- **Matrix leg testing nothing / version-string coercion** — `Not examined: none.` The `analyze` matrix `include` has exactly one leg, `language: actions` + `build-mode: none` (lines 26-28); no duplicated legs, no `exclude`, and no unquoted numeric-looking value that YAML could coerce (`actions`/`none` are plain strings). `build-mode: none` is the required mode for the CodeQL `actions` language. Verified by reading.
- **Version drift between jobs** — `Not examined: none.` `actions/checkout` is pinned to the identical SHA `3d3c42e…` in both jobs (lines 31, 51); the two `codeql-action` steps share one SHA `8aad20d…` (lines 35, 40). No action floats on a tag/branch. Verified by reading.
- **Over-broad permissions** — `Not examined: none.` Top-level is `contents: read` (line 15); each job adds only `security-events: write` (lines 22, 48), which the SARIF uploads genuinely require; `persist-credentials: false` is set on both checkouts (lines 33, 53). Least-privilege is satisfied. Verified by reading.
- **Concurrency cancelling a release** — `Not examined: none.` No `concurrency:` block exists, and this is an analysis-only workflow with no publish/release step, so a cancellation has nothing to interrupt. Verified by reading.
- **Caching correctness** — `Not examined: none.` No `actions/cache` usage and no `cache:` inputs, so there is no cache key to be stale or under-scoped. Verified by reading.
- **Shell assumptions** — `Not examined: none.` There are no `run:` steps; every step is a `uses:` action, so bash-vs-PowerShell quoting semantics do not apply. Verified by reading.
- **Untrusted `pull_request` interpolation into shell** — `Not examined: none.` No `run:` blocks and no `${{ github.event.* }}` interpolation of PR-controlled fields; the only expressions are `${{ matrix.language }}`/`${{ matrix.build-mode }}` (matrix-internal constants defined in-file at lines 27-28). No script-injection sink exists. Verified by reading.

## Did I verify this
- **Read** the full 56-line file — confirmed trigger block, permissions, matrix, SHA pins, `persist-credentials: false`, and absence of `run:`/`if:`/`concurrency:`/`cache:`/`paths:`. All ruled-out entries above are backed by reading and carry `Not examined: none` for the in-file axes they claim.
- **Could not verify** the pinned third-party actions' internal default exit/permission semantics: `github/codeql-action@8aad20d…` and `zizmorcore/zizmor-action@3dc1ecc…` are not local files, and network fetch is unavailable in this session. The single reviewer-relevant consequence of that gap (zizmor's default failure mode) is preserved as the low-confidence finding `CSCAN-codeql-zizmor-gate-may-not-fail` rather than being asserted away.

## Dropped in re-review
None. The sole finding from the first pass is retained at its original low confidence; no ruled-out entry was upgraded, since none had an untried axis that executed evidence could have narrowed (there is no Erlang/OTP code in this file to run, and the unverifiable facts live in remote third-party actions).
