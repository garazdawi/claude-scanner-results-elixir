<!-- rfc-governed: no -->
<!-- standards: -->

# Standards obligation review — `elixir/.github/workflows`

This folder contains GitHub Actions CI/CD configuration and release-automation scripts for the Elixir language project. Its files are:

- `ci.yml`, `codeql.yml`, `markdown.yml`, `posix_compliance.yml` — CI pipelines (build/test, CodeQL scan, markdown lint, a POSIX-shell compliance check job).
- `license_compliance.yml` + `ort/action.yml` — OSS Review Toolkit (ORT) SPDX/license-scanning.
- `release.yml`, `release_pre_built/action.yml`, `release_notifications.yml`, `notify.exs` — release build, packaging, and announcement (GitHub API, Postmark email API, Elixir Forum API).

None of these files **implement** a network protocol, wire format, or cryptographic algorithm. They *consume* HTTPS endpoints (GitHub, Postmark, Discourse) as an opaque client via the `req` library, and verify already-published `.shaNsum` checksum files as opaque strings — they neither define the transport nor compute the digest algorithm themselves. The `posix_compliance.yml` job invokes an external POSIX shell-conformance check on the project's scripts, but that is generic OS-API/secure-coding guidance (POSIX.1 / shell portability), which the per-file security review already covers directly — it is explicitly **not** a protocol/algorithm RFC obligation per the gating rules. `codeql.yml` and `license_compliance.yml` are security/compliance *tooling*, not protocol implementations.

Because the only applicable "standards" here are CI conventions, SPDX license identifiers (a metadata tagging convention, not a wire/algorithm spec), and generic secure-coding/OS-API practice, the RFC-governance verdict is **no** and the external-standards list is empty. The declared-standards input for this application tree was likewise empty, so there is no external entry to account for.

## Applicable standards / contracts

### External standards
*(none)* — This folder does not implement any RFC / ITU-T / W3C / ISO-IEC / NIST / FIPS protocol, wire format, or algorithm. SPDX license tags, GitHub Actions YAML schema, and the third-party HTTP APIs called are consumed as clients, not implemented, and carry no on-the-wire MUST/MUST-NOT conformance clauses a downstream compliance pass could adjudicate here.

### OTP-internal contracts (per-file context, NOT compliance obligations)
*(none)* — no Erlang/OTP modules, `-spec`/`@doc` shapes, or behaviour callbacks are present. The Elixir script (`notify.exs`) is a standalone `Mix.install` automation script, not an OTP component.

## Checkable sub-claims

Although no external protocol/algorithm standard governs this folder, the following supply-chain and secrets-handling obligations are the historically-violated ones worth a per-file glance (they are CI-hygiene / secure-coding checks, reportable under the ordinary security-review classes, not RFC-conformance findings):

1. **ACTIONS_PINNED_BY_SHA** — Third-party `uses:` actions should be pinned to a full commit SHA, not a mutable tag/branch. (Supply-chain integrity; `codeql.yml`, `markdown.yml`, `release*.yml`, `ort/action.yml`.)
2. **SECRETS_FROM_ENV_ONLY** — API tokens (`ELIXIR_LANG_ANN_TOKEN`, `ELIXIR_FORUM_TOKEN`, `X-Postmark-Server-Token`, GitHub token) must come from secrets/env, never hardcoded, and must not be echoed/logged. (`notify.exs`, `release_notifications.yml`.)
3. **MINIMAL_WORKFLOW_PERMISSIONS** — Each workflow should declare least-privilege `permissions:` (e.g. `contents: read`) rather than inheriting broad defaults. (`markdown.yml` sets it; check the others.)
4. **NO_CREDENTIAL_PERSIST** — `actions/checkout` should set `persist-credentials: false` where the job does not push. (Present in `markdown.yml`; verify across jobs.)
5. **CHECKSUM_VERIFIED_NOT_JUST_ECHOED** — Downloaded release checksums (`.shaNsum`) should be *validated* against artifacts, not merely reformatted into an announcement. (`notify.exs` only reprints them — confirm verification happens elsewhere in the release path.)
6. **HTTPS_ENDPOINTS_ONLY** — All external API calls (GitHub, Postmark, forum) must use `https://` URLs. (`notify.exs`, `release_notifications.yml`.)
7. **UNTRUSTED_INPUT_NOT_INTERPOLATED** — Release/tag/`release["body"]` values from the GitHub API must not be interpolated into shell `run:` steps unsanitized (script-injection). (`notify.exs`, `release.yml`.)
8. **PR_TRIGGER_NO_SECRET_EXPOSURE** — `pull_request` triggers must not expose release/publish secrets to fork PRs. (`ci.yml`, `markdown.yml`.)
9. **SPDX_HEADER_PRESENT** — Each file should carry a correct `SPDX-License-Identifier` header for the ORT license scan to pass. (License-compliance convention, all files.)

## Per-file checking instructions

Treat this folder as CI/CD infrastructure, **not** as protocol/algorithm code — the downstream RFC-compliance pass has nothing to adjudicate here (external-standards marker is empty). For each file, use the sub-claims above only as ordinary secure-coding/supply-chain checks: for each claim that could plausibly apply to the file in hand, look for code that satisfies it and name the satisfying construct (e.g. a `permissions:` block, a SHA-pinned `uses:`, `persist-credentials: false`, a `System.fetch_env!` secret read). If a claim would be expected to hold for that file and no satisfying code is present — an unpinned third-party action, a hardcoded token, a broad default permission, or untrusted API text interpolated into a shell step — report it as a finding under the appropriate supply-chain / secrets-handling / injection failure-mode class, not as an RFC-conformance gap.