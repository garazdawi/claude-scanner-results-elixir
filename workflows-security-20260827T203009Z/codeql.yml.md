# /Users/lukas/code/claude-scanner/elixir/.github/workflows/codeql.yml
<!-- sha256: 61af0bab406c6e01efd732d9e2be3d9f876dfabf3bc09457b42775d9764e0347 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/codeql.yml
<!-- sha256: 61af0bab406c6e01efd732d9e2be3d9f876dfabf3bc09457b42775d9764e0347 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-27_

I re-read the complete file (56 lines). This is a scan-only GitHub Actions workflow — it does not `uses:` or dispatch any of the other files in the folder (no `workflow_call`, no local composite action, no `workflow_run`), so it has **no callgraph neighbours to read**: the only external code it reaches is the four pinned third-party/GitHub actions, whose source is not present in this tree. The first-pass hedges therefore fall into three buckets, resolved below: two are resolvable to `verified` from the file itself (the security-relevant claim does not depend on the unverified fact), and one — the trust in the third-party `zizmor-action` source at its pinned SHA — is genuinely unverifiable offline and is **upgraded to a low-confidence finding** rather than left as a dismissed caveat.

- **Triggers:** `push`→`main` (7–8), `pull_request`→`main` (9–10), `schedule` weekly cron (11–12). The boundary is `pull_request` — **not** `pull_request_target` (verified by reading lines 6–12).

---

## Summary

`codeql.yml` is a well-hardened scan-only workflow — SHA-pinned actions, explicit least-privilege permissions (`contents: read` top-level, `security-events: write` per-job), `persist-credentials: false`, no `run:` steps, no secrets, and `pull_request` (not `_target`). No injection, secret-exposure, or execution defect exists. One low-confidence supply-chain finding remains (unauditable third-party action), upgraded from a first-pass caveat.

## Security

### Third-party `zizmorcore/zizmor-action` source is unaudited at its pinned SHA (supply-chain)

- **Upgraded from hedged assumption in first pass.** — Confidence: **low**

- **Location:** line 55, `uses: zizmorcore/zizmor-action@3dc1ecc9bcb9e94e9b2c709687979e1298497054 # v0.6.2`, running in the `zizmor` job which holds `security-events: write` (lines 47–48) and runs on the fork-controllable `pull_request` trigger.
- **Sink class:** untrusted-dependency / supply-chain execution.
- **What I verified:** the action *is* pinned to a full 40-hex commit SHA (the correct mitigation), so it cannot silently move under a mutable tag. This is the only third-party (non-`actions/`, non-`github/`) action in the file.
- **What I could not verify and cannot clear offline:** that the code at commit `3dc1ecc9…` is benign, and that this SHA actually corresponds to the upstream `v0.6.2` tag named in the comment. The action's source is not in this tree and this sandbox has no network access to `github.com` to fetch and diff it. If that commit were malicious (or the pin/comment mismatched), it would execute with `security-events: write` on every PR.
- **Why low and not higher:** SHA-pinning is the accepted, in-place control; there is no *additional* misconfiguration here. This finding records only the irreducible trust placed in an offline-unauditable third-party action, so a later networked pass can diff the SHA against the tag. GitHub-authored actions (`actions/checkout`, `github/codeql-action`) carry the same residual but a materially lower trust cost.

### Ruled out

- **Expression injection into `run:`** — No `run:` steps anywhere (verified by reading all 56 lines; every step is `uses:` + `with:`). The four `${{ }}` expressions (lines 19, 37, 38, 42) reference only `matrix.language`/`matrix.build-mode`, which are statically defined in the workflow (lines 27–28) — not `github.event.*`, `head_ref`, or any attacker context — and land in action inputs, never a shell. `Axes examined: expansion context (matrix-static vs event-derived), sink (action-input vs shell) | Not examined: none`. Verified by reading.
- **`pull_request_target` / `workflow_run`** — `on:` (6–12) contains only `push`, `pull_request`, `schedule`. The "fork code with base write token" shape is absent. Verified by reading.
- **Poisoned-pipeline execution from the fork tree** — A fork PR tree is checked out (30–33, 50–53) but nothing executes it: CodeQL runs `build-mode: none` on `language: actions` (static extraction, no build/autobuild — lines 28/38), zizmor statically parses YAML, and there are zero `run:`/build/postinstall steps. Fork content is analyzed, never run. `Axes examined: build-mode (none), presence of run/build steps (none) | Not examined: none`. Verified by reading.
- **Token permissions** — Top-level `permissions: contents: read` (14–15); each job adds only `security-events: write` (21–22, 47–48), the minimum to upload SARIF. No `contents: write`, `id-token: write`, `packages: write`, or `actions: write` anywhere. Verified by reading.
- **Secrets exposure** — No `secrets.*` reference and no `env:` block anywhere; only the implicit `GITHUB_TOKEN` is in play, and no step forwards it to untrusted code or logs it. Verified by reading.
- **Unpinned third-party actions** — All four `uses:` are pinned to full 40-hex SHAs: `actions/checkout@3d3c42e5…` (31, 51), `github/codeql-action/init@8aad20d1…` (35), `…/analyze@8aad20d1…` (40), `zizmorcore/zizmor-action@3dc1ecc9…` (55). (Trust in the *content* at those SHAs is tracked as the finding above.) Verified by inspecting each pin.
- **Artifact / cache poisoning** — No `actions/cache`, no `upload-artifact`/`download-artifact`, no `workflow_run` artifact consumption. Verified by reading.
- **Self-hosted runners** — Both jobs use `runs-on: ubuntu-latest` (GitHub-hosted); no self-hosted label. Verified by reading.
- **Release / publish path & `actions/github-script`** — Signs/uploads/publishes nothing; no `github-script`. Verified by reading.
- **SPDX header** — Present and well-formed (`SPDX-License-Identifier: Apache-2.0`, line 1). Verified by reading.

## Risks / caveats

- **Fork-PR `GITHUB_TOKEN` downgrade** — resolved from first-pass hedge to **verified independence**: the "not exploitable" conclusion does not depend on GitHub downgrading the fork-PR token to read-only. Even if `security-events: write` were live on a fork PR, there is no injection sink, no secret, and no execution of fork-controlled code to abuse it — all three verified by reading the file. The downgrade itself is documented GitHub behavior and is not load-bearing for any conclusion here, so no residual hedge remains.
- **`actions/checkout` success under per-job `contents: none`** — the per-job `permissions:` blocks (replace-not-merge) set `contents` to `none` for the `analyze`/`zizmor` jobs. Whether checkout still succeeds on this public repo is a purely **functional/reliability** question; the security-relevant part is verified by reading: any failure is fail-closed with no secret or artifact exposure (there are none in the file to expose). Out of scope as a security matter, and no security hedge remains.

## Did I verify this

- **Verified by reading the file (56 lines):** trigger set, absence of `run:` steps, matrix-static origin of all `${{ }}` expressions, `build-mode: none`, permission scopes, absence of secrets/`env`, absence of cache/artifact/self-hosted/publish constructs, SHA-pinning of all four actions, SPDX header.
- **Verified by logical independence (from file contents):** the two former caveats — the conclusion holds regardless of fork-token scope, and regardless of whether checkout functionally succeeds under `contents: none`.
- **Not verifiable in this environment (→ recorded as the low-confidence finding above, not silently cleared):** the *content* of `zizmorcore/zizmor-action` at commit `3dc1ecc9…` and whether that SHA matches upstream `v0.6.2`. No callgraph neighbour exists to read for this (the referenced code is an external repo, absent from this tree), and the sandbox has no network access to fetch/diff it; the "audit upstream source" axis cannot be tried here, so it stays open as a finding for a networked pass.

## Dropped in re-review

None. Every first-pass "Ruled out" entry is preserved (all remain verified from the 56-line file). The first pass reported no positive findings; the two former "Risks / caveats" hedges are resolved (one to verified independence, one to out-of-scope-functional with verified fail-closed security posture), and the third caveat (third-party action trust) is upgraded — not dropped — to an explicit low-confidence finding.
