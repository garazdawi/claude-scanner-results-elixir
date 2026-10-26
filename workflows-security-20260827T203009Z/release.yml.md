# /Users/lukas/code/claude-scanner/elixir/.github/workflows/release.yml
<!-- sha256: 0362a2e6995ebe354250dcc396e59954cac5bbdd82c1f0404d1ef211d8ee08c8 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T20:42:37Z_

# Security review — `release.yml`

## Summary
This release workflow is reachable **only** by pushes to trusted branches/tags and manual `workflow_dispatch` (no `pull_request`/`pull_request_target`/`workflow_run` trigger), so there is no unprivileged-outsider path; I found no outsider-reachable vulnerabilities, only two Low‑severity hardening items.

## Security

The `on:` block (lines 6–15) is `push` to `main`/`v*.*` branches and `v*` tags, plus `workflow_dispatch`. Every one of these requires **write/push access** to the base repository, so the primary attacker in the threat model — an unauthenticated fork‑PR author — cannot fire any of them. Findings below are therefore all "requires push access" and rated accordingly.

#### `CSCAN-release-github-sha-run-interp`
- **Finding ID** — `CSCAN-release-github-sha-run-interp`
- **Severity** — Low
- **Confidence** — high
- **Location** — `release.yml:401` and `release.yml:405`, job `upload-builds-hex-pm`, step "Update builds txt"
- **Class** — Expression injection into `run:`
- **Issue** — `${{ github.sha }}` is substituted directly into a `bash` `run:` script before the shell parses it:
  ```bash
  echo -e "${ref_name}-${otp_version} ${{ github.sha }} ${date} ${build_sha256} \n$(cat builds.txt)" > builds.txt
  ```
  This is the one place in the file that breaks the otherwise‑consistent "read GitHub context through a quoted `$ENV` var" pattern (every `ref_name` use goes through `"$GITHUB_REF_NAME"`).
- **Impact / threat** — Not currently exploitable: `github.sha` is always a 40‑char hex commit id and cannot contain shell metacharacters, and the step runs in the `environment: release` job reachable only by a repo pusher. The concern is pattern hygiene — a copy of this line onto an attacker‑influenceable context (`github.event.*`, `head_ref`, …) would be a direct injection, and it sits next to secret‑bearing AWS/Fastly steps.
- **Evidence** — asserted (read the file; `github.sha` format is a well‑known non‑injectable hex id; not executed).
- **Suggested fix** — route it through `env:` like every other context value in this file:
  ```yaml
      env:
        GIT_SHA: ${{ github.sha }}
      run: |
        ...
        echo -e "${ref_name}-${otp_version} ${GIT_SHA} ${date} ${build_sha256} \n$(cat builds.txt)" > builds.txt
        ...
        echo -e "${ref_name} ${GIT_SHA} ${date} ${build_sha256} \n$(cat builds.txt)" > builds.txt
  ```

#### `CSCAN-release-sign-sbom-contents-write`
- **Finding ID** — `CSCAN-release-sign-sbom-contents-write`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `release.yml:136-139` (job `sign`) and `release.yml:195-199` (job `sbom`)
- **Class** — Token permissions
- **Issue** — Both jobs grant `contents: write`, but neither writes repository contents. `sign` only downloads artifacts, runs `azure/login` + `azure/trusted-signing-action` (needs `id-token: write`), computes hashes, and re‑uploads artifacts (Actions artifact API — needs no `contents`). `sbom` checks out (needs `contents: read`), downloads artifacts, runs ORT, and calls `actions/attest-sbom` (needs `id-token: write` + `attestations: write`). The release *upload* to the repo happens in the separate `upload-release` job, which correctly has `contents: write`.
- **Impact / threat** — Reachable only on trusted push/tag/dispatch. The excess grant widens blast radius if any pinned action in these two jobs were ever compromised at its pinned SHA: a signing/SBOM step could push to the repo. Low because all actions are SHA‑pinned and the trigger requires push access.
- **Evidence** — asserted (reasoned from each step's documented permission needs; not executed, so a transitive need can't be fully excluded).
- **Suggested fix** — drop the write grant to least privilege:
  ```yaml
  # sign job
    permissions:
      id-token: write
  # sbom job (keeps read for actions/checkout)
    permissions:
      contents: read
      id-token: write
      attestations: write
  ```

### Ruled out
- **`pull_request_target` / `workflow_run` fork‑code execution** — none present; the entire `on:` block (lines 6–15) is `push` (branches `main`,`v*.*`; tags `v*`) + `workflow_dispatch`, all requiring repo write access. *Verified by reading the trigger block.*
- **`GITHUB_REF_NAME`/`ref_name` injection into `run:`** — every use is through the quoted default env var (`"$GITHUB_REF_NAME"`, lines 41, 44, 53, 368, 389) or bash parameter expansion (line 350), never a bare `${{ github.ref_name }}` inside a `run:`. The value is passed as data, not source; additionally the `v*`/`v*.*` filters plus git's ref‑name rules bar spaces and `/`. Controlling it also requires push access. *Verified by reading each usage.*
- **Unpinned third‑party actions** — all `uses:` are pinned to a full commit SHA with a version comment (`actions/checkout`, `actions/upload-artifact`, `actions/download-artifact`, `azure/login`, `azure/trusted-signing-action`, `actions/attest-sbom`); the other two are local `./.github/workflows/*` composite actions. *Verified by reading every `uses:`.*
- **Secret exposure to untrusted code** — no fork trigger exists, so no fork‑controlled tree runs with secrets. Release secrets (Azure, `HEX_AWS_*`, `HEX_FASTLY_*`) sit only in `environment: release` jobs (`sign`, `upload-builds-hex-pm`); the `build` job that compiles code has no secrets and only `contents: read`. *Verified by reading the `env:`/`environment:` blocks.*
- **Artifact / cache poisoning across a trust boundary** — no `actions/cache`; artifact hand‑off (`build`→`sign`→`sbom`→`upload-release`) is entirely within one trusted run, and every `download-artifact` omits `run-id`/`github-token`, so it pulls only the current run's artifacts. *Verified by reading the artifact steps.*
- **Self‑hosted runners** — all jobs use GitHub‑hosted `ubuntu-24.04`/`windows-2022`. *Verified.*
- **`actions/github-script` injection** — no `github-script` step in the file. *Verified.*
- **`create_draft_release` credential‑persisting checkout (line 46, `artipacked` ignore)** — deliberate: that job needs `git push origin "$ref_name"` (line 64), and no untrusted code runs between checkout and the git commands; the two other checkouts (lines 87, 210) set `persist-credentials: false`. *Verified by reading the job's steps.*
- **Fork abusing the publish steps** — `upload-builds-hex-pm` fails fast without `vars.HEX_AWS_REGION` (line 331) and the Fastly flush is gated on `github.repository == 'elixir-lang/elixir'` (line 417); a fork lacks the `environment: release` secrets regardless. *Verified by reading the guards.*

## Risks / caveats
- The two local composite actions `./.github/workflows/release_pre_built` (line 90) and `./.github/workflows/ort` (line 222) are **out of scope** and not reviewed here. Note that `github.ref_name` flows into ort's `version:` input at line 225 (`${{ github.ref_type == 'tag' && github.ref_name || github.sha }}`); whether that is safe depends on `ort/action.yml`, which a per‑file reviewer of that file must check.
- Permission‑necessity claims in `CSCAN-release-sign-sbom-contents-write` are **asserted** from each action's documented needs, not from a live run; I could not execute the workflow to observe that `contents: write` is genuinely unused.
- The "no `ref_name` shell injection" conclusion leans on git's ref‑name rules and the `v*`/`v*.*` trigger globs constraining the string (no spaces, `/`, `..`, `*`, control chars). A future change to the trigger filters, or to how `ref_name` is consumed (e.g. a new bare `${{ github.ref_name }}` inside a `run:`), would reopen this — worth re‑checking on any edit to the triggers or the `upload-builds-hex-pm` shell.

