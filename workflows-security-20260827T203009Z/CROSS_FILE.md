# Cross-file findings

## Asserted-dependency ledger

None of the per-file reports use the literal `* Depends on:` syntax; the dependencies below were extracted from each report's "Risks / caveats", "Did I verify", and cross-reference prose (the machine-extractable equivalents). Every cross-file / external assumption is accounted for.

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| release.yml | ort/action.yml — `github.ref_name → version:` input (line 225) is safe | Confirmed | ort-action.yml.md raises it as `CSCAN-action-version-sed-injection` (latent Low, push-gated), not dismissed. |
| release.yml | release_pre_built & ort composite actions ("out of scope") | Confirmed | Both have their own reports with findings; nothing silently trusted. |
| release.yml | `download-artifact` pulls only current-run artifacts (no cross-workflow) | Confirmed | release.yml.md Ruled-out "Artifact/cache poisoning" verified omission of run-id/github-token. |
| ort/action.yml | license_compliance.yml — caller passes `version: github.sha`, `contents: read`, no secrets | Confirmed | license_compliance.yml.md verifies all three by reading. |
| ort/action.yml | release.yml `sbom` job holds `id-token`/`attestations`/`contents: write` + artifact chain | Confirmed | release.yml.md & release_pre_built-action.yml.md both describe the sbom job perms and chain. |
| ort/action.yml | GitHub Actions cache branch-scoping isolates fork writes | Unresolved | Unexecutable platform invariant; file itself upgraded it to a low-conf finding rather than assert. |
| license_compliance.yml | ort/action.yml — mutable `ort-config@main`, `runner.os` cache key, `ort:92.2.0` tag | Confirmed | ort-action.yml.md files all three as findings. |
| license_compliance.yml | `github.sha` is always 40-hex (no sed/shell metachars) | Unresolved | Documented GitHub semantics, asserted, not executed; no report substantiates. |
| license_compliance.yml | fork `pull_request` yields read-only token + no secrets | Unresolved | Platform default, asserted; not verifiable from any report. |
| release_pre_built/action.yml | ci.yml & release_notifications.yml pin `setup-beam` @ v1.24.1 | Confirmed | ci.yml.md (lines 49,130) and release_notifications.yml.md (line 24) show `54075bcc… # v1.24.1`. |
| release_pre_built/action.yml | release.yml artifact chain build→sign→sbom→upload | Confirmed | release.yml.md and release_pre_built-action.yml.md both trace the `needs:` chain. |
| release_pre_built/action.yml | hex.pm must be compromised to deliver GITHUB_ENV payload | Unresolved | External-dependency precondition, asserted; keeps severity Low. |
| ci.yml | some other workflow consuming `TestCoverage` artifact via `workflow_run` | Unresolved | No report checks a consumer; no `workflow_run` trigger exists in-folder (release.yml pulls only own-run artifacts). |
| ci.yml | fork `pull_request` read-only token + no secrets | Unresolved | Platform default, asserted. |
| ci.yml | `ex_doc@main` upstream is benign at run time | Unresolved | Mutable ref executed; external repo not in tree — raised as `CSCAN-ci-exdoc-unpinned-main-exec`. |
| release_notifications.yml | **notify.exs** sanitizes `$GITHUB_REF_NAME` / API release body before URL/shell use | **Unresolved** | **No per-file report exists for notify.exs**; sanitization unchecked by anyone. |
| release.yml / THREAT_MODEL | **notify.exs** interpolates `System.argv()` into URLs/shell safely | **Unresolved** | Same missing-report coverage gap; RFC_OBLIGATIONS §5/§7 also expects notify.exs checks. |
| release_notifications.yml | `GITHUB_REF_NAME` is runner-provided / not shadowable via step `env:` | Unresolved | Platform behavior, asserted; not executed. |
| codeql.yml | `zizmor-action@3dc1ecc9…` content matches upstream `v0.6.2` and is benign | Unresolved | Offline-unauditable external SHA; file upgraded it to a low-conf finding. |
| markdown.yml | `markdownlint-cli2-action` at pinned SHA loading fork `.cjs/.mjs`/customRules | Unresolved | Action source absent from tree; dismissal rests on verified containment, axis un-runnable. |
| markdown.yml | repo-root `.markdownlint-cli2.jsonc` content | Unresolved | File not present in scanned tree; cannot be read. |
| posix_compliance.yml | shellcheck does not execute analyzed scripts (`-x` off) | Unresolved | Tool property asserted (shellcheck not installed); moot given verified containment. |

## Findings

The ledger has **no Contradicted rows** — every asserted cross-file dependency that another report *does* address is Confirmed, and where reports agree they agree consistently (e.g. the `github.sha`-vs-`ref_name` trust split matches the THREAT_MODEL's primary/secondary attacker distinction). The Unresolved rows are coverage gaps (nobody checked), not defects, and per the instructions are left as ledger rows only. One legitimate Step-2 systemic pattern rises to a folder-level finding.

#### `CSCAN-crossfile-mutable-upstream-in-signed-release-chain`
- **Finding ID** — `CSCAN-crossfile-mutable-upstream-in-signed-release-chain`
- **Severity** — Medium
- **Confidence** — high
- **Files involved** — `release.yml.md`, `release_pre_built-action.yml.md`, `ort-action.yml.md`, `ci.yml.md`, `THREAT_MODEL.md`
- **Issue** — Each per-file report flags its own single mutable-upstream execution, but no report tallies the aggregate: the **signed, SBOM'd, attested release** produced by `release.yml` depends on at least three unpinned upstream sources pulled by two different composite/build steps, all feeding artifacts that later get `id-token`/`attestations`/`contents: write` treatment in the `sign`/`sbom` jobs. (1) `release_pre_built/action.yml:58` fetches `ex_doc`'s `latest_stable_version` from hex.pm and checks that ref out (`CSCAN-action-hexpm-github-env-injection`), and its `build` output flows build→sign→sbom→upload. (2) `ort/action.yml:47` checks out `oss-review-toolkit/ort-config@main` and (3) `ort/action.yml:94` runs `ort:92.2.0` by mutable tag (`CSCAN-action-ort-config-mutable-ref`, `CSCAN-action-ort-image-mutable-tag`), both feeding the attested SPDX SBOM in `release.yml`'s `sbom` job (`release.yml:222`). Separately `ci.yml:82` executes `ex_doc@main` on every push/PR. Individually each is rated Medium/Low; collectively they mean a single upstream compromise (hex.pm, `ort-config`, or the ORT registry — the THREAT_MODEL's "tertiary attacker") lands inside the trusted signing boundary, and the SBOM/attestation then vouches for the poisoned bytes. The cross-file point the per-file lane cannot make: the signing trust boundary in `release.yml:35` is only as strong as the *weakest* of these three unpinned upstreams, not any one in isolation.
- **Resolution** — Pin all three to reviewed digests/SHAs (hex.pm `ex_doc` version → validated pin per `release_pre_built`'s own suggested fix; `ort-config@main` → commit SHA; `ort:92.2.0` → `@sha256:` digest) so the signed-release chain has no mutable-upstream dependency. No source re-read needed — all three lines are already verified in the cited reports; this finding only aggregates their shared blast radius onto the signing boundary.

## Coverage gap of note (ledger, not a finding)

`notify.exs` is invoked by `release_notifications.yml:34` and referenced by `release.yml` and `THREAT_MODEL.md` as the place where `$GITHUB_REF_NAME` and untrusted `api.github.com` release-body/`browser_download_url` content are interpolated into URLs and shell — and RFC_OBLIGATIONS §5/§6/§7 explicitly expects checksum-verification, HTTPS-only, and no-untrusted-interpolation checks there. **No per-file report for `notify.exs` exists in this report set.** Both `release_notifications.yml.md` and `release.yml.md` correctly hand the risk off to it ("out of scope, must be audited in notify.exs directly") rather than clearing it by assertion, so there is no contradiction to mint — but the sanitization those two files depend on has been verified by nobody. This is the single highest-value follow-up: obtain/produce a `notify.exs` review before relying on the release-notification path being injection-safe.
