<!-- rfc-governed: no -->
<!-- standards: -->

# Documented-contract review: `elixir/.github/workflows`

## What this folder is

This is the CI/CD configuration directory for the Elixir language repository. It contains GitHub Actions workflow definitions (`ci.yml`, `codeql.yml`, `license_compliance.yml`, `markdown.yml`, `posix_compliance.yml`, `release.yml`, `release_notifications.yml`), an OSS Review Toolkit (`ort/`) config subtree, a pre-built release helper subtree (`release_pre_built/`), and one Elixir script (`notify.exs`) that posts release announcements to an email service (Postmark) and the Elixir Forum via HTTP JSON APIs. There are **no `.erl` files and no `*.hrl` headers** in this folder; there is no protocol, wire-format, or cryptographic-algorithm implementation here.

## Applicable standards / contracts

### External standards
None. No file in this folder implements a network protocol, wire format, document format, or cryptographic algorithm defined by an external standard. `notify.exs` is an HTTP *client* of the GitHub, Postmark, and Elixir Forum REST APIs — it consumes those vendor JSON APIs but does not define or implement a standardized wire format whose byte-level correctness a downstream third-party consumer depends on. The workflow YAML files are consumed only by GitHub Actions (a build-orchestration config format, not a protocol/algorithm standard). Names like `posix_compliance.yml` refer to a *test job the CI runs against the Elixir codebase*, not to POSIX obligations governing these config files themselves. The mechanically-gathered standards list for the application tree was empty, consistent with this finding.

### OTP-internal contracts (per-file context, NOT compliance obligations)
None apply — there is no Erlang/OTP module code here (no `-spec`, no `@doc`, no `gen_*` behaviour callbacks). `notify.exs` is a plain top-level Elixir script.

## Checkable sub-claims

These are secure-coding / robustness obligations for the script and workflow config, **not** external protocol-conformance obligations. They are already within the remit of the per-file security review; listed here only so a per-file reviewer knows what to look for.

1. SECRET_NOT_LOGGED — Tokens read via `System.fetch_env!/1` (`ELIXIR_LANG_ANN_TOKEN`, `ELIXIR_FORUM_TOKEN`, Postmark server token) must not be echoed into `IO.puts`/step output or committed. (`notify.exs`.)
2. HTTP_STATUS_CHECKED — Responses from external POSTs should have their status inspected rather than a failure being silently swallowed; a non-2xx from Postmark/Forum should be observable. (`notify.exs`.)
3. DRAFT_RELEASE_GUARDED — Announcements must not fire for a draft/unpublished release (the `if release["draft"]` guard) — releasing prematurely is the observable failure. (`notify.exs`.)
4. DRYRUN_HONORED — Side-effecting POSTs must remain gated behind the `DRYRUN` env check so test/dry runs do not send real email or forum posts. (`notify.exs`.)
5. WORKFLOW_PERMS_LEAST_PRIVILEGE — Workflows should declare minimal `permissions:` (e.g. `contents: read`) rather than defaulting to broad write scopes. (`ci.yml` and siblings.)
6. ACTION_PINNED_BY_SHA — Third-party actions should be pinned to an immutable commit SHA (as `actions/checkout@3d3c42…`, `erlef/setup-beam@54075b…` already are), not floating tags, to resist supply-chain tampering. (workflow YAMLs.)
7. CREDENTIALS_NOT_PERSISTED — Checkout steps handling code should set `persist-credentials: false` where the token is not needed downstream (already present in `ci.yml`). (workflow YAMLs.)
8. UNTRUSTED_INPUT_NOT_INTERPOLATED — `pull_request`-triggered workflows must not interpolate attacker-controllable fields (PR title/body/ref) directly into `run:` shell blocks (script injection). (workflow YAMLs.)
9. RELEASE_ASSET_CHECKSUM_INTEGRITY — When `notify.exs` fetches and republishes per-asset checksums, it must reproduce the upstream checksum verbatim without truncation or re-encoding. (`notify.exs`.)

## Per-file checking instructions

For each file, treat the sub-claims above as secure-coding checks, **not** as external-standard conformance items — the RFC/compliance pass has nothing to adjudicate here (verdict `no`, empty standards list). For each sub-claim that could plausibly apply to the file in hand, search for the code or config that satisfies it: if a workflow declares a least-privilege `permissions:` block, SHA-pins its actions, or sets `persist-credentials: false`, name it as satisfying the claim; if `notify.exs` guards side effects behind `DRYRUN` and the draft check, name those guards. If a workflow interpolates untrusted `pull_request` fields into a shell `run:` step, or a secret is echoed to output, or a side-effecting POST is ungated, report it as a finding under the appropriate injection / secret-exposure / unchecked-side-effect class. Do not raise "RFC/standard non-conformance" findings for anything in this folder.