# /Users/lukas/code/claude-scanner/elixir/.github/workflows/release_notifications.yml
<!-- sha256: 1fc7fad4fd01985bc18f036f3114cfaa76414abd2f12a3bcc6936599df845983 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T20:41:42Z_

Verification confirms the file is clean on both key axes:

- **Grep**: the only two `${{ }}` expressions in the file are the `secrets.*` reads in the `env:` block (lines 31–32). Neither lands in a `run:` script. The `run:` step references the runner env var `$GITHUB_REF_NAME`, not `${{ github.ref_name }}`.
- **Bash**: a double-quoted parameter expansion of a value containing `$(touch …)` and `` `touch …` `` passed the payload through as a literal `argv[1]` and created no files — confirming the shell layer here does not re-evaluate the tag value.

Here is the report.

---

## Summary

`release_notifications.yml` is a well-hardened release-triggered workflow — least-privilege token, SHA-pinned actions, credential-less checkout, and a correctly env-quoted argument — with no security defects found in this file.

## Security

`No findings`

### Ruled out

Search space: the complete file, all 35 lines read; the sole trigger is `release: { types: [published] }` (lines 6–9), which on GitHub can only be fired by an actor with write/maintain/admin permission publishing a release — an unprivileged fork-PR/issue/comment outsider cannot reach any sink here. Each class below was checked against the full file.

- **Expression injection into `run:`** — *Verified, ruled out.* Grep shows the only `${{ }}` substitutions are `secrets.ELIXIR_FORUM_TOKEN` / `secrets.ELIXIR_LANG_ANN_TOKEN` in the `env:` block (lines 31–32), the safe location. The single `run:` (line 34) is `elixir .github/workflows/notify.exs "$GITHUB_REF_NAME"`, which uses the runner-provided env var `$GITHUB_REF_NAME` — double-quoted — rather than `${{ github.ref_name }}`. My bash test showed a tag value of `v1.0$(touch …)` `` `touch …` `` is passed as a literal `argv[1]` with no command execution and no files created. This is exactly the recommended env-and-quote mitigation, already applied.
- **`pull_request_target` / `workflow_run`** — *Ruled out.* The `on:` block (lines 6–9) declares only `release: published`. Neither high-risk trigger is present, so there is no fork-controlled-checkout-with-write-token shape. Verified by reading the whole trigger block.
- **Token permissions** — *Ruled out (positive control present).* A workflow-level `permissions: { contents: read }` (lines 11–12) is declared and not overridden by the `notify` job, so `GITHUB_TOKEN` is read-only. This is the least-privilege baseline; nothing here requests `contents: write`, `id-token: write`, or `packages: write`.
- **`secrets` exposure** — *Ruled out.* The two API tokens are read from the `secrets` context into `env:` scoped to the single "Run Elixir script" step (lines 30–34); they are not placed on the `setup-beam` step, not echoed, and not passed to a third-party action. The step executes the in-repo `notify.exs`, which — because the trigger is `release: published`, not a fork PR — is trusted base-repo code at the released ref. `persist-credentials: false` on checkout (line 22) additionally prevents the git-stored token from being left on disk for that step to harvest.
- **Unpinned third-party actions** — *Verified, ruled out.* Both `uses:` are pinned to full 40-hex-char commit SHAs: `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1` (line 20) and the third-party `erlef/setup-beam@54075bcc5e249e4758d363f27d099f55d843f124` (line 24). No mutable tag/branch refs.
- **Artifact / cache poisoning** — *Ruled out.* No `actions/cache`, `actions/upload-artifact`, or `download-artifact` steps exist in the file; nothing is consumed across a trust boundary.
- **Self-hosted runners** — *Ruled out.* `runs-on: ubuntu-latest` (line 16) is a GitHub-hosted, ephemeral runner.
- **Release / publish path reachability** — *Ruled out.* This workflow only *announces*; the publish/version input `$GITHUB_REF_NAME` is the tag of an already-published release, and firing the `release: published` trigger itself requires write access. The value is passed safely (see above), so a crafted tag cannot influence this file's shell execution.
- **`actions/github-script`** — *Ruled out.* No `github-script` step is present; there is no JavaScript injection surface in this file.

## Risks / caveats

- **`notify.exs` is out of scope and is where the real risk lives.** This file passes `$GITHUB_REF_NAME` safely-quoted, but the folder threat model notes `notify.exs` interpolates `System.argv()` (this same value) into URLs and shell, and trusts `api.github.com` release JSON / `browser_download_url` bodies. Whether the tag name or fetched release body is sanitized there is not verifiable from this file and must be audited in `notify.exs` directly. Note the attacker for that path is a tag-pusher/maintainer (write access), since only they can publish a release.
- **Third-party action trust.** `erlef/setup-beam` is SHA-pinned (good), but pinning fixes *which* code runs, not *what that code does*; the pinned revision is trusted to install OTP/Elixir without tampering. It does not receive the announcement secrets (they are scoped to the later step), which bounds its blast radius.
- **`GITHUB_REF_NAME` provenance.** I relied on the documented GitHub Actions behavior that `GITHUB_REF_NAME` for a `release` event is the release's tag and is runner-provided (not settable via the step `env:`), so it cannot be shadowed here. I did not execute the workflow to confirm the runtime value.

