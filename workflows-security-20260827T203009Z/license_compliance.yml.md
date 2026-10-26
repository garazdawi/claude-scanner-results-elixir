# /Users/lukas/code/claude-scanner/elixir/.github/workflows/license_compliance.yml
<!-- sha256: c7e82cbe0c383362f0ec5fd77d03f94d317542caf02f3b431078fffaf8141908 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/license_compliance.yml
<!-- sha256: c7e82cbe0c383362f0ec5fd77d03f94d317542caf02f3b431078fffaf8141908 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-27_

I read the full file (42 lines) and, this pass, the full neighbour composite action `ort/action.yml` (113 lines) that this file invokes at line 36. My conclusion is unchanged: no defect lives in this file. The one first-pass hedge that a neighbour read could resolve ("I could not verify" the `ort/action.yml` weaknesses) is now verified by reading that file.

---

## Summary

`license_compliance.yml` is a well-hardened read-only CI scan — explicit least-privilege `permissions: contents: read`, no `pull_request_target`, SHA-pinned checkout with `persist-credentials: false`, no secrets, and its single `${{ }}` interpolation (`github.sha`, line 41) is passed as a composite-action **input**, never into a `run:` step in this file. No finding in this file.

## Security

No findings.

Triggers are `push`, `pull_request`, `workflow_dispatch` (lines 6–9). Only `pull_request` is reachable by an unprivileged fork author; on that path `GITHUB_TOKEN` is read-only and no repository secrets are exposed, and this file further pins to `contents: read` (lines 11–12). Every sink below was checked against that outsider-reachable path.

### Ruled out

- **Expression injection into `run:`** — Ruled out, **verified** by reading all 42 lines. Exactly one `run:` step exists (line 26, `git config --global url...insteadOf`) and it contains no `${{ }}` interpolation. The only interpolation, `"${{ github.sha }}"` (line 41), is passed as the `version` input to the composite action, not into any shell in this file. Confidence: High.
  - Axes examined: presence of `${{ }}` inside every `run:` in-file (one `run:`, no interpolation); the single interpolation's sink (composite input, not shell). Not examined: none for this file's `run:` steps.
- **Downstream injection of `github.sha` in the composite** — Ruled out, **verified** by reading `ort/action.yml`. The `version` input lands in `ELIXIR_VERSION` (line 81) and is expanded inside `sed`/`mkdir`/`cp` shell in the composite (lines 72, 77–79). `github.sha` is a Git object hash (`[0-9a-f]{40}`) and cannot carry shell/`sed` metacharacters, so even the downstream expansion is not an injection vector. That `github.sha` is always 40-hex is documented GitHub semantics — **asserted** (not executable here). Confidence: Medium (the format guarantee is asserted, not run).
- **`pull_request_target` / `workflow_run`** — Ruled out, **verified**. The `on:` block (lines 6–9) declares only `push`, `pull_request`, `workflow_dispatch`. No "write token + secrets over fork code" shape. Confidence: High.
- **Token over-privilege** — Ruled out, **verified**. Top-level `permissions:` grants only `contents: read` (lines 11–12). No `id-token: write`, `contents: write`, `packages: write`, `pull-requests: write` anywhere. The composite inherits this scope and cannot escalate (confirmed: `ort/action.yml` declares no `permissions:` of its own — a composite cannot widen the caller's grant). Confidence: High.
- **Secrets exposure** — Ruled out, **verified**. No `secrets.` reference in this file; none placed in `env:` or passed to the composite. Reading `ort/action.yml` confirms the composite requests no secrets and sets `persist-credentials: false` on its own `ort-config` checkout (line 49), so no credential is planted for fork-controlled steps. `persist-credentials: false` here (line 32) keeps the checkout token off disk. Confidence: High.
- **Unpinned third-party actions (this file)** — Ruled out, **verified**. The only third-party action, `actions/checkout`, is SHA-pinned `3d3c42e5...` (line 30). The other `uses:` is `./.github/workflows/ort` (line 36), a first-party local composite for which SHA-pinning does not apply. Confidence: High.
- **Self-hosted runner** — Ruled out, **verified**. `runs-on: ubuntu-24.04` (line 20) is GitHub-hosted/ephemeral. Ephemerality is documented platform behavior — **asserted**. Confidence: High.
- **Release / publish path** — Ruled out, **verified**. This job runs a license scan and (via the composite) uploads scan reports only; reading `ort/action.yml` confirms there is no sign/registry-upload/publish of build artifacts — the `upload-results` ORT step (line 104) uploads scan reports, gated on `upload-reports == 'true'`. The scanner version is `github.sha`, not an attacker-steerable payload value. Confidence: High.
- **Poisoned-pipeline execution (PPE) via scanning fork code** — Ruled out as an *in-file* defect. On `pull_request` the fork-controlled merge tree is checked out and ORT scans it, so fork-authored build metadata is evaluated by the scanner. This file applies the correct containment — read-only `GITHUB_TOKEN`, no secrets, `persist-credentials: false`, ephemeral runner — so blast radius is generic compute. Intended hardened configuration, not a defect here. Containment basis: **verified** from lines 11–12, 20, 32; "no secrets on fork `pull_request`" is documented GitHub behavior — **asserted**. Confidence: High that this file is correctly contained; Medium that the platform property holds (asserted).

## Risks / caveats

- **The composite action `./.github/workflows/ort` (line 36) is where the untrusted PR tree is actually processed, and it carries two supply-chain weaknesses — now confirmed by reading `ort/action.yml`, not assumed:**
  - It checks out `oss-review-toolkit/ort-config` at **`ref: "main"` — an unpinned mutable ref** (`ort/action.yml` lines 46–47). **Upgraded from hedged assumption in first pass** ("I could not verify them here"): now **verified by reading the file**.
  - Its ScanCode `actions/cache` is keyed on **`${{ runner.os }}-scancode` alone** (`ort/action.yml` line 88), a broad cache key. **Verified by reading the file.**
  - Additionally observed while reading: the ORT runtime container is pinned by **tag, not digest** — `image: ghcr.io/oss-review-toolkit/ort:92.2.0` (line 94), while the `ort-ci-github-action` wrapper itself is SHA-pinned (line 92).
  - None of these three weaknesses live in `license_compliance.yml`; this file is only the trigger that reaches them on `pull_request`. They should be reported against `ort/action.yml` directly, and are noted here as neighbour context only. Confidence: High (read directly).
- Claims resting on GitHub platform semantics rather than file contents — that fork `pull_request` yields a read-only token with no secrets, that `github.sha` is always 40-hex, that GitHub-hosted runners are ephemeral — are **asserted** from documented behavior. I did not execute the workflow and cannot try these axes in the sandbox (they are platform-runtime properties, not local Erlang/shell behavior), so they remain asserted.
- I did not inspect the ORT container image contents or whether ORT's Mix/SPDX analyzer executes project code during dependency resolution. If it does, the (correctly contained) PPE surface above is real but bounded by this file's read-only, secret-free configuration.

## Did I verify this?

- **Verified by reading** (this file, 42 lines): single `run:` step and its lack of interpolation; the `github.sha` input sink; `on:` triggers; `permissions: contents: read`; absence of `secrets.`; `persist-credentials: false`; SHA-pinned checkout; `runs-on: ubuntu-24.04`.
- **Verified by reading the neighbour** `ort/action.yml` (113 lines, this pass): `ort-config@main` unpinned ref (lines 46–47); ScanCode cache keyed on `runner.os` (line 88); container pinned by tag not digest (line 94); wrapper action SHA-pinned (line 92); composite has no `permissions:` block and requests no secrets; `github.sha` reaches `sed`/shell but only as a 40-hex value.
- **Asserted** (documented GitHub semantics, not executed — no axis available to try in-sandbox): fork `pull_request` token/secret scoping; `github.sha` format guarantee; runner ephemerality.

## Dropped in re-review

None. Every ruled-out entry from the first pass is retained. The only substantive change is upgrading the `ort/action.yml` caveat from "I could not verify them here" to **verified by reading `ort/action.yml`**, and recording that those weaknesses belong to that neighbour file, not this one.
