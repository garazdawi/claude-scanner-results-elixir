# Threat Model — `elixir/.github/workflows` (GitHub Actions CI/CD)

> **Scope note:** this folder contains **no `.erl` files**. It is the Elixir language project's CI/CD supply chain: seven GitHub Actions workflows (YAML), two local composite actions (`ort/`, `release_pre_built/`), and one Elixir script (`notify.exs`). The threat model below is for CI/CD supply-chain security, not a wire protocol. Governing standards: **GitHub Actions security-hardening guidance**, **SLSA** (build provenance), **Sigstore/in-toto attestations**, **SPDX/CycloneDX** (SBOM), and **OpenSSF Scorecard / zizmor** conventions (pinned actions, least-privilege tokens).

## Entry points (trigger surfaces)
- `ci.yml` — `push`, `pull_request`, `workflow_dispatch`. Compiles and tests on Linux/Windows across OTP matrix.
- `release.yml` — `push` to `main`/`v*.*`, `v*` tags, `workflow_dispatch`. Builds, **signs**, SBOMs, and **publishes** releases to GitHub, S3 (hex.pm), and purges Fastly.
- `release_notifications.yml` → `notify.exs` — `release: published`. Posts to a Postmark mailing list and Elixir Forum.
- `codeql.yml`, `license_compliance.yml`, `posix_compliance.yml`, `markdown.yml` — lint/scan on push/PR.
- Composite actions `ort/action.yml`, `release_pre_built/action.yml` — invoked by the above.

## Trust / contract boundaries
- **`pull_request` from forks** is the sharpest boundary. `ci.yml`, `codeql.yml`, `license_compliance.yml`, `markdown.yml`, `posix_compliance.yml` all run **attacker-authored code/config** (a fork PR fully controls the checked-out tree, `Makefile`, test suite, `bin/elixir`). These run under `permissions: contents: read` with the default `GITHUB_TOKEN` and **no secrets** passed — the intended containment.
- **Secret-bearing jobs are gated on trusted triggers**: `notify.exs` (Postmark `ELIXIR_LANG_ANN_TOKEN`, `ELIXIR_FORUM_TOKEN`), the `sign`/`upload-builds-hex-pm` jobs (`environment: release`, Azure signing creds, `HEX_AWS_*`, `HEX_FASTLY_*`) run only on `release`/tag/`main` events, never on fork PRs. The `environment: release` gate is a load-bearing control.
- **`tag`/`ref_name` is attacker-influenceable-ish**: `notify.exs` interpolates `System.argv()` (`$GITHUB_REF_NAME`) straight into URLs and shell; `release.yml` interpolates `GITHUB_REF_NAME` into `gh`/`aws`/`sed`/`tag` commands. Whoever can push a tag controls these strings.
- **Third-party network fetches**: `notify.exs` trusts `api.github.com` release JSON and `browser_download_url` bodies; `release_pre_built` `curl`s `hex.pm` and `jq`-parses it into `$GITHUB_ENV`; `ort` checks out `oss-review-toolkit/ort-config@main` (**unpinned mutable ref**) and runs a pinned-by-digest ORT container.

## Attacker model
- **Primary: unauthenticated fork-PR author** (pre-auth relative to the repo). Goal: poisoned-pipeline execution to steal secrets or the write-scoped `GITHUB_TOKEN`, or tamper with build artifacts.
- **Secondary: a maintainer/tag-pusher** who can influence `ref_name` and thereby the publish/notify paths.
- **Tertiary: a compromised third-party dependency** — an action, the ORT container, `ort-config@main`, `ex_doc`, or a fetched checksum file.
- **Out of scope (implicitly):** GitHub platform compromise, malicious core maintainers with `environment: release` access.

## Common attack patterns for this family (recognition cues)
- **Script injection** via `${{ github.* }}` expanded inside `run:` — check every `run:` that references `github.event.*`, `ref_name`, `github.sha`. (`release.yml` uses `${{ github.sha }}` directly inside `run:` blocks, lines ~401/405.)
- **Poisoned pipeline execution (PPE)** — fork PR altering build/test scripts that later run with secrets.
- **`pull_request_target` / secret leakage to forks** — none present here (good); verify no future drift.
- **Unpinned actions / mutable refs** — most actions are SHA-pinned with version comments; exceptions: `ort-config@main` and `setup-beam` pinned to an **older SHA (v1.18.2)** in `release_pre_built` vs v1.24.1 elsewhere.
- **`GITHUB_ENV`/`GITHUB_PATH` injection** — `release_pre_built` writes `curl|jq` output and `$PWD/bin` into them (flagged with `# zizmor: ignore[github-env]`).
- **Artifact poisoning across jobs** — `build`→`sign`→`sbom`→`upload` pass artifacts by name; a compromised earlier job taints signed output.
- **Token over-privilege** — check per-job `permissions:` escalations (`contents: write`, `id-token: write`, `attestations: write`, `security-events: write`).
- **Cache poisoning** — `actions/cache` for ScanCode keyed only on `runner.os`.

## "Process"/structure & job-graph invariants
There are no BEAM gen_servers/supervisors here; the analogue is the **job dependency graph** in `release.yml`: `create_draft_release`, `build` → `sign` → `sbom`, then `upload-release` needs `[create_draft_release, build, sign, sbom]`. Signing happens **after** build in a separate `environment: release` job — the trust boundary that keeps signing keys away from untrusted build steps.

## Cross-file invariants the per-file reviewer must not lose
1. **Secrets never reach fork PRs**: relies on secret-bearing jobs being gated by trigger (`release`/tag) and `environment: release`. A reviewer must not clear a `run:` interpolation in `ci.yml` by assuming secrets are present — they aren't; conversely must not clear one in `release.yml`/`notify.exs` by assuming the input is trusted — `ref_name` is not fully trusted.
2. **`persist-credentials: false`** is set on nearly every `checkout` — later steps assume no git-credential is left on disk. The one exception (`artipacked` ignore in `release.yml` line 47) is deliberate.
3. **`notify.exs` idempotency/guard**: it `raise`s on draft releases and honors `DRYRUN`; the workflow assumes the script itself enforces "don't double-post."
4. **Composite actions inherit the caller's token/secrets and `permissions`** — `ort` and `release_pre_built` have no `permissions:` block of their own; their blast radius is whatever the calling job granted (`sbom` grants `id-token`/`attestations: write`).
5. **ORT config provenance**: `ort/action.yml` mutates `$HOME/.ort/config` from a mutable upstream `@main` plus in-repo `.ort/` files; downstream SBOM/attestation integrity assumes that merged config is trustworthy.