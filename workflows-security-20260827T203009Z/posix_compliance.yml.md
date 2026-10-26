# /Users/lukas/code/claude-scanner/elixir/.github/workflows/posix_compliance.yml
<!-- sha256: 057c5447d2b636c1b7ac44263edb83f6788de06e0acb6878162acbe8036ad674 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T20:39:26Z_

I have enough to write the report. Every containment fact is verified directly from the file: no `${{ }}` expressions (grep), no `secrets.*` references, no `pull_request_target`/`workflow_run`, no self-hosted runner, no write scopes, SHA-pinned checkout with `persist-credentials: false`, and top-level `contents: read`.

# Security review — `posix_compliance.yml`

## 1. Summary

A well-hardened, lint-only workflow that runs `shellcheck` (a non-executing static analyzer) against three checked-out shell scripts with a read-only token, no secrets, no write scopes, and a SHA-pinned checkout — no meaningful security defect found.

## 2. Security

`No findings`

- **Triggers present:** `push` (path-filtered to `.github/workflows/posix_compliance.yml`, `bin/elixir`, `bin/elixirc`, `bin/iex`), `pull_request` (same path filter — fireable by any unprivileged outsider via a fork PR touching those paths), and `workflow_dispatch` (maintainers only). The `pull_request` path is the outsider-reachable surface and is the one I weighted most heavily below.

### Ruled out

- **Expression injection into `run:`** — *verified.* Grep for `${{` across the whole file returns no matches, and the full 54-line read confirms it: every `run:` step uses static, hardcoded commands and hardcoded filenames (`bin/elixir`, `bin/elixirc`, `bin/iex`). No `github.event.*`, `head_ref`, or any context value is substituted into a shell line, so there is no injection sink on any trigger.
- **Poisoned pipeline execution via `checkout` + build/lint of fork code** — the `pull_request` run does check out fork-controlled `bin/elixir`/`elixirc`/`iex` (a real outsider-controlled tree), but the only thing done with them is `shellcheck`, which is a static parser/linter that builds an AST and reports diagnostics — it does not evaluate or execute the scripts it analyzes, and external-source following (`-x`/`--external-sources`, the only mode that even *reads* other referenced files) is not enabled. So fork content is inspected, never run. This tool property I state as *asserted* (shellcheck is not installed in this environment, so I could not run a live demo); however, the *containment* that makes it moot is *verified* from the file: the job holds only `permissions: contents: read` (line 20-21), references no `secrets.*` (grep: no matches), and runs on ephemeral GitHub-hosted `ubuntu-latest` (line 29) — even a hypothetical execution would get a read-only token, no secrets, and no persistence.
- **`pull_request_target` / `workflow_run` secret exposure** — *verified.* Grep for `pull_request_target` and `workflow_run` returns no matches; the trigger block (lines 7-18) is plain `pull_request` + `push` + `workflow_dispatch`. Fork PRs therefore run with the default read-only token and no secrets, not a write token in the base context.
- **Token over-privilege** — *verified.* Top-level `permissions: contents: read` (lines 20-21) applies to the single job; grep confirms no `id-token: write`, `packages: write`, or `contents: write` anywhere. Least-privilege baseline is met.
- **Secrets exposure** — *verified.* No `secrets.*` reference exists in the file (grep: no matches); nothing is echoed, written to the workspace, or passed to a step, because there are no secrets in scope.
- **Unpinned third-party actions** — *verified.* The only `uses:` is `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1` (line 35), pinned to a full 40-hex commit SHA. No mutable tag/branch action is present.
- **Credential persistence** — *verified.* The checkout sets `persist-credentials: false` (line 37), so no git credential is left on disk for the fork-controlled tree that follows.
- **Self-hosted runner** — *verified.* `runs-on: ubuntu-latest` (line 29); no `self-hosted`/`runs-on:` label pointing at persistent infrastructure (grep: no `self-hosted` match).
- **Artifact / cache poisoning** — *verified by absence.* The file (read in full) declares no `actions/cache`, no `upload-artifact`/`download-artifact`, and no `workflow_run` artifact consumption; there is no cross-trust-boundary storage to poison.
- **Release / publish path** — *verified by absence.* This workflow only installs shellcheck and lints; it contains no sign/upload/publish step and, with `contents: read` and no secrets, could not publish even if reached. The tag/version cannot be influenced into a publish here because there is no publish.

## 3. Risks / caveats

- The "shellcheck does not execute the analyzed scripts" property is **asserted**, not verified in-environment (shellcheck is not installed on this host, so I could not run a malicious-input demo). It is a documented property of the tool and `-x` is not enabled, and — more importantly — the finding is moot regardless, because the job's read-only token, absence of secrets, and ephemeral runner (all file-verified) cap the blast radius of any hypothetical execution.
- Review was scoped to this single file. The `bin/elixir`/`elixirc`/`iex` scripts being linted, the `actions/checkout` action at the pinned SHA, and the upstream Ubuntu `shellcheck` package are trusted as-is; their contents were not audited here.
- A GitHub-platform-level compromise (malicious `ubuntu-latest` image or apt mirror) is out of scope and unchanged by anything in this file.

