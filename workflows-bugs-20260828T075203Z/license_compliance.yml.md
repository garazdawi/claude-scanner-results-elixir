# /Users/lukas/code/claude-scanner/elixir/.github/workflows/license_compliance.yml
<!-- sha256: c7e82cbe0c383362f0ec5fd77d03f94d317542caf02f3b431078fffaf8141908 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/license_compliance.yml
<!-- sha256: c7e82cbe0c383362f0ec5fd77d03f94d317542caf02f3b431078fffaf8141908 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

# License Compliance Workflow Review

## Summary
`license_compliance.yml` is a small, correctly-gated workflow — least-privilege `contents: read`, SHA-pinned checkout, `persist-credentials: false`, and an explicit `fail-on-violation: true` that makes this the one workflow whose ORT invocation actually blocks on license/policy violations. No silent-failure bug lives in this file itself. On re-review I read the sibling composite action `ort/action.yml` (the target of every "out of scope" hedge in the first pass) and confirmed the three deferred claims: the `fail-on` gate does consume this file's input correctly, the SHA interpolation is metacharacter-safe, and the ScanCode cache key genuinely lacks a lockfile hash — the last of which I now surface as a low-confidence finding because it directly weakens this workflow's blocking guarantee.

## Bugfixes

No findings originating inside this file.

## Risks / caveats

- **Low-confidence finding — ScanCode cache key has no content hash, can serve stale scan results to the one blocking license check** (Confidence: low). **Upgraded from hedged assumption in first pass.** The first pass wrote "its `${{ runner.os }}-scancode` cache key … lives there, not here — out of scope." I read `ort/action.yml:84-88` and confirmed: `key: ${{ runner.os }}-scancode` (line 88) contains only the OS string — no lockfile hash, no dependency digest, no `version`/SHA component. `restore-keys` is absent, but the exact key itself never changes across commits, so every run on a given OS restores whatever ScanCode result set was first cached. Because `license_compliance.yml` is the sole call site that passes `fail-on-violation: true` (line 39 → gates `fail-on: "…violations,issues…"` at `ort/action.yml:105`), a stale/poisoned ScanCode cache can cause a genuine violation to be missed while the job still reports green — the failure mode this workflow exists to prevent. The defect physically resides in `ort/action.yml:88`, not in this file; it is reported here at low confidence because (a) this file is the only path where the missed-violation consequence is load-bearing, and (b) I could not execute the runner to confirm the runtime cache-hit behaviour, only read the key definition. Fix belongs in `ort/action.yml` (add a hash of the ORT config / lockfiles to the key).

- I could not execute GitHub Actions; the expression-evaluation rulings below are reasoned from documented GitHub Actions semantics plus the now-read `ort/action.yml` source, not from an observed run. There is no Erlang/OTP code on this path, so the "try the axis in the sandbox toolchain" route does not apply — GitHub Actions expression evaluation cannot be reproduced with the local Erlang toolchain.

### Ruled out

- **`fail-on-violation: true` silently not gating** — *Neighbour now read (`ort/action.yml:105`), no longer hedged.* Line 105 is `fail-on: "${{ inputs.fail-on-violation == 'true' && 'violations,issues' || '' }}"`, exactly the string comparison the first pass reasoned about. This file passes the YAML boolean `true` (line 39). Composite-action `inputs` are always stringified by the runner (typed inputs exist only for `workflow_call` reusable workflows; `ort/action.yml` is `using: "composite"`, line 40, so it is not that case), therefore `inputs.fail-on-violation` is the string `"true"` and `'true' == 'true'` → true → `fail-on` receives `violations,issues`. The gate fires. Same reasoning clears `upload-reports: true` → `upload-results` at line 104. Confidence in the ruling: high, but derived from GitHub's documented input-stringification contract confirmed against the read source, not from an executed run — I cannot run a GitHub Actions expression engine locally, so this is not claimed as execution-verified.

- **`version: "${{ github.sha }}"` interpolation corrupting `sed` regexes** — *Neighbour now read, verified.* From this call site `inputs.version` = `github.sha`, which `ort/action.yml:81` binds to `ELIXIR_VERSION`, consumed by four `sed` commands: line 72 (`-E`, `/` delimiter, value in replacement), line 77 (`/` delimiter), line 78 (`-E`, `/` delimiter), line 79 (`|` delimiter). A commit SHA is `[0-9a-f]{40}` — it contains no `/`, no `|`, and no regex/replacement metacharacter — so from *this* call site none of those four substitutions can be corrupted. (Note: the other caller `release.yml` passes a tag/ref as `version`, where `/` in the ref is a live concern — but that is a different call site and belongs to a `release.yml` review, not this file.) Confidence: high.

- **Least-privilege permissions** — `permissions: { contents: read }` (lines 11–12) is minimal; no job requests write or `id-token`. Satisfies WORKFLOW_PERMS_LEAST_PRIVILEGE. Not a finding.

- **Action pinning** — `actions/checkout` (line 30) is pinned to full SHA `3d3c42e5…` (v7.0.1). The two transitive `uses:` in `ort/action.yml` (checkout `11bd7190…` line 44, cache `d4323d4d…` line 85, ort-ci-github-action `086d928d…` line 92) are likewise full-SHA-pinned — verified while reading the neighbour. No floating tags on this path. Satisfies ACTION_PINNED_BY_SHA.

- **Credential persistence / git-config rewrite** — The `url.https://github.com/.insteadOf ssh://git@github.com/` rewrite (line 26) runs before a checkout with `persist-credentials: false` (line 32), so no `GITHUB_TOKEN` is embedded in `.git/config`; the rewrite direction (ssh→https) only affects dependency clones. Not a leak on this `contents: read` workflow. Satisfies CREDENTIALS_NOT_PERSISTED.

- **Trigger/filter excluding target files** — `on:` has `push`, `pull_request`, `workflow_dispatch` with **no** `paths:`/`branches:` filters (lines 6–9); nothing narrows away the files a whole-repo license scan must cover. Not a filter bug.

- **Untrusted-input script injection** — This file has no `run:` block that interpolates `${{ github.event.* }}`, PR title/body, or branch name; the only inline `run:` (line 26) is a constant. The single `${{ }}` value it forwards (`github.sha`, line 41) reaches a shell only via `env:` in the neighbour (`ort/action.yml:81`, quoted `"$ELIXIR_VERSION"` usage), not spliced into a `run:` body. Satisfies UNTRUSTED_INPUT_NOT_INTERPOLATED. Not a finding.

- **Duplicate push + pull_request runs** — For a same-repo branch with an open PR, both triggers fire two identical runs: redundant CI cost, not a silent failure (no check stops running). Outside this review's failure model.

## Did I verify this?

- I **read** `ort/action.yml` in full and confirmed line-by-line the three claims the first pass had marked "out of scope": the `fail-on` gate (line 105), the SHA→`sed` dataflow (lines 81, 72/77/78/79), and the ScanCode cache key (line 88). Those hedges are resolved.
- I did **not** execute a GitHub Actions run, and the sandbox's Erlang/OTP toolchain cannot stand in for the GitHub Actions expression evaluator — so the input-stringification ruling and the cache-hit behaviour are reasoned from documented runner semantics plus the read source, not from an observed run. That residual uncertainty is why the cache-key item is filed as *low* confidence rather than dismissed, and why the two expression-based dismissals note they are contract-derived rather than execution-verified.

## Dropped in re-review

None. Every ruled-out entry from the first pass is preserved; two of them ("fail-on gate" and "version interpolation") are strengthened from hedged-and-deferred to neighbour-read-and-verified, and the ScanCode-cache caveat is upgraded from an out-of-scope aside to an explicit low-confidence finding.
