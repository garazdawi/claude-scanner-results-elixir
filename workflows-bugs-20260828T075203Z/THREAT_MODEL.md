# Folder Bug/Risk Profile: `elixir/.github/workflows`

**Note for the reviewer:** this folder contains **no `.erl` files**. It is the GitHub Actions CI/CD pipeline for the Elixir language repository — nine YAML workflow/action definitions plus one Elixir release-notification script (`notify.exs`). The "protocol family" here is **GitHub Actions / OIDC supply-chain security**, not an Erlang wire protocol. Treat this as a *build-and-release pipeline* review: the failure modes are supply-chain, secret-exposure, and script-injection, not `function_clause` crashes.

## What this code does
CI/CD automation governed by the **GitHub Actions workflow schema**, GitHub's OIDC/attestation model (Sigstore, SLSA provenance), SPDX/CycloneDX **SBOM** standards, and RFC 3161 timestamping (Azure Trusted Signing). Entry points ("triggers"):
- `ci.yml` — push/PR/dispatch; compiles Elixir across OTP matrix, runs test suites.
- `release.yml` — push to `main`/`v*.*`/tags, dispatch; builds, **signs**, generates SBOM+attestations, uploads to GitHub Releases, S3 (hex.pm), Fastly purge.
- `codeql.yml` / `license_compliance.yml` / `posix_compliance.yml` / `markdown.yml` — scanners/linters.
- `release_notifications.yml` → `notify.exs` — on `release: published`, posts to Postmark email + Elixir Forum.
- `ort/action.yml`, `release_pre_built/action.yml` — composite actions.

## Trust / contract boundaries
- **Attacker-influenceable inputs:** `pull_request`-triggered workflows run with attacker-authored code. The key defense is that `ci.yml`, `codeql.yml`, `markdown.yml`, `posix_compliance.yml` hold only `contents: read` (and scoped `security-events: write`), and set `persist-credentials: false`. Verify any newly-added step in a PR-triggered workflow does not gain write scope or handle secrets.
- **`notify.exs` boundary:** `tag` comes from `$GITHUB_REF_NAME` (a *published release* — maintainer-controlled, low risk), but `release["body"]` and asset names/checksums come from the **GitHub API response** and are interpolated into forum posts / email `HtmlBody`. `release["assets"]` checksum bodies are fetched over the network and string-split without validation — malformed asset content yields a wrong-but-non-crashing announcement (silent corruption), or a `MatchError` on `[checksum | _]`/`"." <> type` for unexpected names.
- **Secrets exposed at boundaries:** `release.yml`/`license_compliance.yml` set `GITHUB_TOKEN`, AWS keys, Fastly keys, Azure client IDs, Postmark/Forum tokens. The `sign`/`upload-builds-hex-pm` jobs use `environment: release` and `id-token: write` (OIDC) — these are the crown-jewel jobs.
- **Operator-controlled config:** OTP version matrices, hex.pm AWS vars, Azure signing vars. Note `AZURE_TRUSTED_SIGNING_ACCOUNT_NAME` gates whether Windows binaries are actually signed — if unset, unsigned `.exe`s still flow to upload.

## Failure-mode model (this pipeline's equivalents)
- **Script injection** — untrusted context (`${{ github.event.* }}`, PR titles, branch names) interpolated directly into `run:` shell. Most steps here correctly pass values via `env:` and use `"$VAR"`; flag any direct `${{ }}` inside `run:` bodies.
- **`git-config` credential leak** — `git config --global url.https://…insteadOf ssh://…` (used in `license_compliance.yml`, `release.yml` sbom job) rewrites *all* clones.
- **Unpinned actions** — supply-chain: all third-party actions here are SHA-pinned (good); a reviewer should treat any tag/branch-pinned `uses:` as a finding.
- **Over-broad permissions** — a job granting `contents: write`/`id-token: write` where read suffices.
- **`notify.exs`** — `MatchError`/`badmatch` on unexpected API shapes; missing `draft` guard is present, but no handling of HTTP non-200 from asset downloads.

## Common bug patterns for this family (reviewer cues)
- `pull_request_target` misuse (not present — confirm it stays absent).
- Cache poisoning via `actions/cache` keyed only on `runner.os` (see `ort/action.yml` `scancode` cache — no lockfile hash in key: stale/poisonable cache).
- `sed -i` interpolating `${ELIXIR_VERSION}`/`${GITHUB_REF_NAME}` into regexes (`ort/action.yml`, `release.yml`) — a ref name with `/` or regex metacharacters corrupts substitutions.
- Artifact name/`pattern:` glob mismatches between upload (`sign-*`) and download (`{sign-*-elixir-otp-*,…}`) jobs → silently empty release.
- `continue-on-error` on `development` matrix rows masking real regressions.

## Process / job structure (the safety boundaries)
`release.yml` is a DAG: `create_draft_release` + `build` → `sign` → `sbom` → `upload-release`; `upload-builds-hex-pm` depends on `build`+`sign`. `sign` and the two hex/upload jobs are the trust-critical nodes (OIDC token, signing keys, AWS/Fastly creds). A compromise or logic bug *before* `sign` (in `build`/`release_pre_built`) propagates a tampered artifact into the signed, attested output — the attestation would then vouch for malicious bytes.

## Cross-file invariants
- `release.yml`'s `sbom`/`upload` jobs assume `release_pre_built/action.yml` produced exactly `elixir-otp-${otp}.zip`/`.exe`; the S3 upload `sed`/glob logic in `upload-builds-hex-pm` hard-depends on that naming.
- `license_compliance.yml` and `release.yml` both call `./.github/workflows/ort`; the ORT action's `fail-on-violation` default (`false`) means license checks only *block* in `license_compliance.yml` (which sets it `true`) — the release path does **not** fail on license violations.
- `notify.exs` assumes `release_notifications.yml` only fires on a real published (non-draft) release; its `raise "cannot notify a draft release"` is the sole guard.
- `posix_compliance.yml`/`markdown.yml` use `paths:` filters — they do **not** run on unrelated changes, so they are not a safety net for the bin/ scripts in most PRs.