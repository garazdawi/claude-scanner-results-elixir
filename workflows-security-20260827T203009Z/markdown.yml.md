# /Users/lukas/code/claude-scanner/elixir/.github/workflows/markdown.yml
<!-- sha256: c9e3d78985031c712bd25c8211fc835a0ce29c3acf0b685c9c0fb3f0b1180837 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/markdown.yml
<!-- sha256: c9e3d78985031c712bd25c8211fc835a0ce29c3acf0b685c9c0fb3f0b1180837 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-27_

# Markdown Content workflow — security review (re-review)

## 1. Summary

`markdown.yml` is a tightly-hardened lint-only workflow: explicit top-level least-privilege token (`contents: read`), both actions SHA-pinned, `persist-credentials: false`, no `run:` shell steps, no `${{ }}` interpolation, and no secrets referenced anywhere in the file. Re-reading the full 42-line file confirms every containment claim from the first pass. **No findings.**

The first-pass hedges concerned two things: (a) whether `markdownlint-cli2` will execute a fork-supplied JS config/custom-rule module, and (b) the content of the external linter action and of `.markdownlint-cli2.jsonc`. I could not read the neighbours for either: the scanned tree at `/Users/lukas/code/claude-scanner/elixir/` contains **only** `.github/` — no repo-root `.markdownlint-cli2.jsonc` exists (`ls` + `find` confirm), and the pinned third-party action's source is an external SHA not present in the tree. Neither fact is verifiable here. I resolve the hedges below by (i) confirming the code-execution axis is a genuinely un-runnable external axis (so the entry stays `asserted`, not falsely promoted to `verified`), and (ii) confirming the dismissal does not *rely* on that axis — it rests on the verified "nothing to steal" containment, which holds whether or not fork code runs.

## 2. Security

`No findings`

Triggers: `push` to `main` (path-filtered, line 7–14), `pull_request` (path-filtered via the `*paths-filter` anchor, line 16–17), and `workflow_dispatch` (line 19). Only `pull_request` is reachable by an unprivileged outsider (a fork PR touching a `.md` file, `markdown.yml`, or `.markdownlint-cli2.jsonc`); `push`/`main` requires merge and `workflow_dispatch` requires write access. Each outsider-reachable path was checked against every Phase 1 sink class below.

### Ruled out

- **Expression injection into `run:`** — none possible. There is no `run:` step and no `${{` context interpolation anywhere in the file (verified by re-reading all 42 lines; both steps are `uses:` actions, lines 36 and 40–41). No shell sink exists regardless of trigger.
  - Axes examined: presence of `run:` steps, presence of `${{ }}` expansion, both triggers. | Not examined: none.

- **`pull_request_target` / `workflow_run`** — absent. The only fork-reachable trigger is plain `pull_request` (line 16), which runs with the top-level read-only token and no secrets — the safe variant, not the write-token-over-fork-code shape.
  - Axes examined: full `on:` block (lines 6–19). | Not examined: none.

- **Token permissions** — least privilege is explicit. `permissions: contents: read` is declared at top level (lines 21–22). The single `lint` job (line 28) has no `permissions:` override, so it inherits read-only and cannot escalate. No `id-token`, `packages: write`, `contents: write`, `attestations`, or `security-events` appears anywhere.
  - Axes examined: top-level `permissions`, per-job overrides. | Not examined: none.

- **`secrets` exposure** — none. No `secrets` context is referenced. The only `env:` is `LANG: C.UTF-8` (lines 24–25). Nothing sensitive is placed in the environment, a file, or a step input, so even arbitrary code execution on the runner has nothing to exfiltrate.
  - Axes examined: `env:` block, all step inputs, whole-file `secrets` reference. | Not examined: none.

- **Unpinned third-party actions** — both actions are pinned to full 40-hex commit SHAs: `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1` (line 36, `# v7.0.1`) and `DavidAnson/markdownlint-cli2-action@21c1be1b93ad9ed58fa840aacc3f279cde2a72ff` (line 41, `# v24.2.0`). Pinning-by-SHA of the third-party linter is the relevant supply-chain hardening (satisfies **ACTIONS_PINNED_BY_SHA**).
  - Axes examined: both `uses:` refs. | Not examined: whether the pinned SHAs' *code* is itself benign — the action sources are not in the scanned tree (only `.github/` is checked out), so this is unverifiable here and out of scope for a per-file review.

- **Artifact / cache poisoning** — no `actions/cache`, no `upload-artifact`/`download-artifact`, and no consumption of another workflow's output. Nothing crosses a job/workflow trust boundary.
  - Axes examined: all steps. | Not examined: none.

- **Self-hosted runners** — `runs-on: ubuntu-latest` (line 30), a GitHub-hosted ephemeral runner. No persistent infrastructure is exposed to fork code.
  - Axes examined: `runs-on`. | Not examined: none.

- **Release / publish paths** — none. This job only lints; no sign, upload, or publish step, and no tag/version derived from the event payload.
  - Axes examined: all steps and `env`. | Not examined: none.

- **`actions/github-script`** — not used.
  - Axes examined: both `uses:` refs. | Not examined: none.

- **SPDX header** — present and well-formed: `SPDX-License-Identifier: Apache-2.0` (line 1). Satisfies **SPDX_HEADER_PRESENT**.
  - Axes examined: file header. | Not examined: none.

- **Fork PR running untrusted content in the linter** — dismissed on **verified impact**, `asserted` on the execution axis. `pull_request` checks out the fork-controlled merge tree, and `markdownlint-cli2` *may* execute attacker-supplied JavaScript (e.g. a `.markdownlint-cli2.cjs`/`.mjs` config or a `customRules` module a PR adds). I treat that code execution as **assumed true (worst case, in the attacker's favour)** rather than relying on it being false — so the dismissal does not depend on the un-runnable execution axis. Even granting full code execution, the containment that makes this a non-finding is verified on this file: the job holds only a `contents: read` token (line 22, no job override), receives no secrets (no `secrets` reference in the file), checks out with `persist-credentials: false` (line 38, so no git credential is left on disk for the attacker's code to read), and runs on an ephemeral GitHub-hosted runner (line 30). On a public repo the read-only token exposes only already-public data. The residual gain to an outsider is throwaway ephemeral compute — the accepted residual risk of linting PR content, and it is hardened correctly (satisfies **PR_TRIGGER_NO_SECRET_EXPOSURE** and **NO_CREDENTIAL_PERSIST**).
  - Axes examined: token scope (verified read-only), secret presence (verified none), credential persistence (verified `false`), runner ephemerality (verified hosted), worst-case impact assuming execution (verified nil). | Not examined: whether `markdownlint-cli2-action` at the pinned SHA actually loads and executes a fork-supplied `.cjs`/`.mjs`/`customRules` module. The action's source is not in the scanned tree (only `.github/` is checked out) and no local Node harness for it exists, so this axis is **un-runnable here**. It is not load-bearing: the entry is dismissed on impact, which holds whether the axis is true or false. Entry remains **asserted** (available for a later pass to re-examine), not promoted to `verified`.

## 3. Risks / caveats

- Only `.github/workflows` is present in the scanned tree; the repo is not fully checked out. The linter action's source (external SHA) and `.markdownlint-cli2.jsonc` (not present — `ls`/`find` confirm no repo-root config) could not be read. Pinning fixes *what* runs but not whether that fixed external code is itself safe; that is out of scope for a per-file review and the source is unavailable here.
- If a future change adds secrets, grants a write permission, or switches the trigger to `pull_request_target`/`workflow_run`, the "nothing to steal" calculus flips and this workflow needs re-review — the fork-PR entry above would then become a live finding rather than accepted residual risk.

## 4. Did I verify this?

- **Re-read the full file (42 lines).** Verified: no `run:` step, no `${{ }}`, no `secrets`, top-level `permissions: contents: read` with no job override, both actions SHA-pinned (lines 36, 41), `persist-credentials: false` (line 38), `runs-on: ubuntu-latest` (line 30), only `env` is `LANG` (line 25), SPDX header present (line 1).
- **Attempted to read the flagged neighbours.** `Read` of `.markdownlint-cli2.jsonc` → does not exist. `ls -la` + `find` of `/Users/lukas/code/claude-scanner/elixir/` → only `.github/` present. The linter action's source is an external SHA not in the tree. Conclusion: the two first-pass hedges point to neighbours that are **not present in the scanned tree** and therefore cannot be read or run here.
- **Un-runnable axis handled honestly.** The "does `markdownlint-cli2` execute fork JS" axis cannot be tried in this sandbox (no action source, no harness), so it stays on `Not examined` and the fork-PR entry stays **`asserted`**, not `verified`. I did not upgrade it to a finding because impact under the worst-case assumption is verified nil (read-only token, no secrets) — creating a "finding" whose exploit yields nothing would be a false positive. Kept as an asserted dismissal so a later pass can revisit if the tree becomes fully available.

### Dropped in re-review

None. All prior ruled-out entries are preserved. The prior report carried no findings, so none were dropped. No hedged assumption was upgraded to a finding: the single reasoned assumption (`markdownlint-cli2` executing fork JS) was restructured into a conservative worst-case *granted* to the attacker, under which the verified containment still yields no finding — a downgrade-safe resolution rather than an upgrade.
