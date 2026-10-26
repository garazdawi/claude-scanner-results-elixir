# /Users/lukas/code/claude-scanner/elixir/.github/workflows/ci.yml
<!-- sha256: fcc50d3f863cd8d1402f22a47bd25aaff18095c035657b3042b1f8280751715f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T20:35:24Z_

# Security review — `ci.yml`

## 1. Summary

A well-hardened CI workflow (`permissions: contents: read`, no secrets, SHA-pinned actions, GitHub-hosted runners, no event-payload interpolation); the only real defect is a docs job that clones and executes `elixir-lang/ex_doc@main` from an unpinned mutable ref.

## 2. Security

- **Triggers:** `push`, `pull_request`, `workflow_dispatch` (lines 8–10). `pull_request` is reachable by any unprivileged outsider via a fork PR; `push`/`workflow_dispatch` require repo write access. All jobs inherit the top-level `permissions: contents: read` (lines 17–18), and no `secrets.*` are referenced anywhere in the file — so fork-PR runs get a read-only token over a public repo and no secrets.

#### `CSCAN-ci-exdoc-unpinned-main-exec`
- **Finding ID** — `CSCAN-ci-exdoc-unpinned-main-exec`
- **Severity** — Low
- **Confidence** — high
- **Location** — `ci.yml:80–87`, job `test_linux`, step "Build docs (ExDoc main)" (clone at line 82; execution at lines 84 and 87)
- **Class** — Unpinned third-party actions (here via `git clone` in `run:` rather than `uses:`)
- **Issue** — The step clones `https://github.com/elixir-lang/ex_doc.git --depth 1` (the mutable tip of `main`), then compiles and runs it: `../elixir/bin/mix do local.rebar --force + local.hex --force + deps.get + compile` (line 84) and `make docs` (line 87). Nothing pins ex_doc to a reviewed commit/tag or verifies a checksum, so whatever `main` points to at run time is executed.
- **Impact / threat** — Runs for the `docs` matrix cell (OTP 28.4) on every `push`, `pull_request` (incl. fork PRs), and `workflow_dispatch`. If ex_doc's `main` is compromised (or its repo/registry/DNS is), attacker code runs on the runner. Containment is strong: `permissions: contents: read`, no secrets defined, ephemeral GitHub-hosted runner (`ubuntu-24.04`) — so there is no secret to steal, the token is read-only over an already-public repo, and there is no persistence. For a fork-PR author this grants nothing beyond what editing their own `Makefile` already does; the only marginal exposure is that an ex_doc compromise would also execute in trusted maintainer `push` builds. Hence Low.
- **Evidence** — verified: read of lines 80–87 shows the unpinned clone-and-execute; `grep` confirms no `ref`/SHA pin and that no secrets or write permissions are in scope for the job.
- **Suggested fix** — Pin to a reviewed revision, e.g.
  ```yaml
  - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
    with:
      repository: elixir-lang/ex_doc
      ref: <reviewed-commit-sha>
      path: ../ex_doc
      persist-credentials: false
  ```
  or, keeping the `git clone`, add `git -C ex_doc checkout <reviewed-commit-sha>` before compiling.

### Ruled out

- **Expression injection into `run:`** — verified: all 13 `${{ }}` occurrences reference only `matrix.*` (author-defined in-file), never `github.event.*` / `github.head_ref` / `*.user.*`; and none land in a `run:` shell body — they appear in `name:`, `if:`, `with:`, `continue-on-error:`, and `env:`. No attacker-controlled event text reaches a shell.
- **`pull_request_target` / `workflow_run`** — verified via grep: neither trigger present, so there is no fork-code-with-write-token shape.
- **Poisoned pipeline execution (fork code runs in CI)** — present by design (`make compile`/`test_erlang`/`test_elixir` build the fork's tree on `pull_request`) but contained, not a defect: top-level `permissions: contents: read` applies to every job, grep shows zero `secrets.*` references, and runners are GitHub-hosted ephemeral (`ubuntu-24.04`, `windows-2022`). A fork gains only a read-only token on a public repo.
- **Token over-privilege** — verified: top-level `contents: read`; no job requests `id-token`/`packages`/`contents: write` or any escalation.
- **`secrets` exposure** — verified via grep: no `secrets.` reference anywhere in the file, so nothing is passed to fork-controlled build steps, logs, or actions.
- **Unpinned `uses:` actions** — verified: all five `uses:` are pinned to full 40-char SHAs (`actions/checkout`, `erlef/setup-beam`, `actions/upload-artifact`).
- **Self-hosted runners** — verified via grep: only `ubuntu-24.04` and `windows-2022`; no `self-hosted` label, so untrusted PR code never lands on persistent infrastructure.
- **Release / publish path** — verified by read: this file only compiles/tests and uploads a `TestCoverage` artifact; it contains no sign/publish/upload-to-registry step and no tag/version derived from the event payload.
- **`actions/github-script`** — verified: not used.
- **`GITHUB_ENV` / `GITHUB_PATH` injection** — verified by read: line 55 writes a static literal; line 60 writes `$PWD/bin` (the runner-determined workspace path, not fork-controlled content). No attacker data flows into either.

## 3. Risks / caveats

- **Cross-workflow artifact provenance (out of this file's scope):** the `TestCoverage` artifact (line 95) is produced on `pull_request` builds too, so its contents can be fork-controlled. Any *other* workflow that later downloads CI artifacts by name — especially via `workflow_run` — must verify which workflow/SHA produced them; I could not check the consuming side from this file.
- **`otp_version: master` / `maint` (lines 37–40)** drive `erlef/setup-beam` to fetch and build OTP from mutable upstream branches — intentional nightly testing under the same read-only/no-secret containment, but it is another unpinned-upstream execution worth noting.
- The "fork PRs get a read-only token and no secrets" reasoning rests on GitHub's platform default for `pull_request`, which is asserted (a platform guarantee), not something enforced within this file.
- Only `ci.yml` was reviewed; interactions with the sibling `release.yml` / `release_notifications.yml` (shared caches, artifact consumption, downstream signing) are out of scope here.

