# /Users/lukas/code/claude-scanner/elixir/.github/workflows/release_notifications.yml
<!-- sha256: 1fc7fad4fd01985bc18f036f3114cfaa76414abd2f12a3bcc6936599df845983 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/release_notifications.yml
<!-- sha256: 1fc7fad4fd01985bc18f036f3114cfaa76414abd2f12a3bcc6936599df845983 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T08:00:52Z_

# Release Notifications Workflow Review

## Summary
`release_notifications.yml` is a clean, least-privilege notification workflow with no conditions, matrices, caches, or concurrency to misbehave. I found no bugfix-worthy defects in it. On re-review I read the one callgraph neighbour this file invokes (`notify.exs`) to convert the first pass's cross-file assumptions into verified statements of the env/argv contract; the neighbour read did not surface any defect *in this file*.

## Bugfixes

No findings.

The file's checkable surface is small and each item holds up:

- **Trigger** — `on: release: types: [published]` correctly matches the intent (announce a published release) and never fires for drafts, so `notify.exs`'s `raise "cannot notify a draft release"` guard (verified at `notify.exs:16-18`) is a redundant defense consistent with the trigger, not a dead-vs-live guard that hides anything.
- **Permissions** — `permissions: contents: read` is minimal and sufficient; the job only checks out the repo and calls external HTTP APIs. Satisfies least-privilege.
- **Action pinning** — `actions/checkout@3d3c42…` and `erlef/setup-beam@54075b…` are both pinned to full 40-hex commit SHAs (comments name the tags v7.0.1 / v1.24.1), resisting tag re-pointing. `persist-credentials: false` is set on checkout.
- **Injection** — the tag reaches the script through the runner-provided env var `$GITHUB_REF_NAME`, quoted (`"$GITHUB_REF_NAME"`), not via `${{ … }}` interpolation into the shell, so there is no template-injection path even if the tag were adversarial.
- **Env/argv contract (verified against `notify.exs`, not assumed)** — the `run:` step passes exactly one positional argument; `notify.exs:6` consumes it as `[tag] = System.argv()`, so the arity matches (one arg in, one-element match). The two secrets the script dereferences with `System.fetch_env!/1` — `ELIXIR_LANG_ANN_TOKEN` used as the Postmark server token at `notify.exs:55`, `ELIXIR_FORUM_TOKEN` at `notify.exs:73` — are both supplied by the `env:` block at lines 31–32. No missing-env crash. Neither token is echoed to step output by this workflow (the `IO.puts` calls at `notify.exs:59,78` print response status/body, not the header tokens).
- **Failure propagation** — the single `run:` step has no `continue-on-error` and no `|| true`; `Req.post!`/`Req.get!` (`notify.exs:14,26,58,77`) raise on non-2xx, and the top-level `%{status: 200, body: …} = Req.get!(…)` match at `notify.exs:13` hard-fails on any non-200, so a failed announcement surfaces as a red job rather than being swallowed.

## Risks / caveats

- **Verification basis is `asserted` (documented-semantics reasoning), not executed, for the GitHub-runtime layer.** GitHub Actions cannot be run in this environment, so trigger/permission/env-injection behavior was reasoned against documented GitHub semantics, not observed. The `notify.exs` env/argv contract above *was* verified by reading the neighbour and is stated as verified; the GitHub-runtime delivery of those values is what remains asserted.
- **`GITHUB_REF_NAME` equivalence to the release tag** rests on GitHub's documented behavior that, for `release` events, `GITHUB_REF` is `refs/tags/<tag>` and `GITHUB_REF_NAME` is that tag name. I confirmed the downstream consumer (`notify.exs:6,14`) would `MatchError` if the tag were empty/non-tag, but that failure mode is a robustness concern of `notify.exs`, not a defect of this YAML — this file correctly forwards a single quoted argument. Using `${{ github.event.release.tag_name }}` would be a marginally more explicit source but is not a fix for a bug in this file.
- **Cross-file scope.** The correctness of what happens *after* the tag is forwarded (checksum parsing at `notify.exs:22-34`, the `"." <> type = Path.extname(name)` match at `notify.exs:32` that can `MatchError` on an extensionless asset name, unchecked bodies from the per-asset `Req.get!` at `notify.exs:26`) lives in `notify.exs`. Reading it here confirmed the env/argv contract; those downstream robustness issues are findings for `notify.exs`'s own review, not for this YAML, and are recorded here only as the reason this file's contract is safe.

### Ruled out
- **Missing `if:` draft guard on the job** — not a defect. `types: [published]` does not fire for drafts, and the script carries its own draft guard (`notify.exs:16-18`, read this pass); adding a workflow-level `if` would be redundant and its absence hides no check. Axes examined: trigger-type semantics, presence of an independent script-level guard | Not examined: none.
- **No `concurrency:` block (possible double-post on duplicate events)** — considered and rejected as a bugfix finding: GitHub does not emit duplicate `published` events for a single publish action, and a cancel-mid-publish concurrency group would risk aborting an in-flight notification rather than help. Basis: documented single-fire semantics of `release: published`. Axes examined: event-fire multiplicity, cancel-in-progress side effects | Not examined: live GitHub duplicate-delivery behavior (cannot execute GitHub Actions in sandbox) — entry remains asserted.
- **Missing `shell: bash` / Windows quoting** — not applicable: `runs-on: ubuntu-latest`, whose default shell is bash, under which `"$GITHUB_REF_NAME"` expands as intended. Axes examined: runner OS, default shell, quoting | Not examined: none.
- **Version drift between jobs** — not applicable: there is exactly one job, and `otp-version`/`elixir-version` are both quoted pinned strings (`"27.3"`, `"1.18.3"`); no second job to diverge from. Axes examined: job count, version pinning | Not examined: none.

## Did I verify this
- **Verified by reading `notify.exs` (this pass):** the env-var/argv arity contract (`notify.exs:6`), that both `env:`-supplied secrets are the ones dereferenced (`notify.exs:55,73`), that the script has its own draft guard (`notify.exs:16-18`), and that this workflow does not echo either token. These upgrade the first pass's "out of scope / outside this file" hedges to verified statements of the contract.
- **Asserted (not executable here):** GitHub Actions runtime delivery of `release: published` events, the `GITHUB_REF_NAME` → tag mapping, and single-fire event semantics. GitHub Actions cannot be run in this sandbox, so these remain documented-semantics reasoning.

### Upgraded from hedged assumption in first pass
- The first pass hedged that the env contract ("the two secrets the script actually dereferences … are both supplied by the `env:` block") and the argv forwarding were reasoned about `notify.exs` from "outside this file." This pass read `notify.exs` and confirmed both directly (`notify.exs:6,55,73` against YAML lines 31–34), so these are now stated as **verified**, not assumed. No new finding results, because the confirmed behavior is correct and this file forwards its inputs correctly.

### Dropped in re-review
None. Every conclusion from the first pass is preserved; the cross-file items were verified rather than dropped.
