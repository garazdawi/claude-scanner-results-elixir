# /Users/lukas/code/claude-scanner/elixir/.github/workflows/markdown.yml
<!-- sha256: c9e3d78985031c712bd25c8211fc835a0ce29c3acf0b685c9c0fb3f0b1180837 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/markdown.yml
<!-- sha256: c9e3d78985031c712bd25c8211fc835a0ce29c3acf0b685c9c0fb3f0b1180837 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

# Markdown Content workflow review — `markdown.yml`

## Summary
The workflow is small, correctly SHA-pinned, least-privilege, and its triggers/path filters correctly cover the files it lints. The one hedge in the first pass — that the linter's effective file set lives in `.markdownlint-cli2.jsonc`, which the first pass "could not read" — has now been resolved: that config file is **not present anywhere in the reviewed tree** (only `.github/` was extracted). It is therefore genuinely unreadable, not merely out of scope, so the assumption "it presumably defines a covering `globs`" cannot be verified and is upgraded to a low-confidence finding below rather than left as a caveat.

## Bugfixes

### Silent no-op lint if `.markdownlint-cli2.jsonc` lacks a covering `globs` — **Upgraded from hedged assumption in first pass**
- **Location:** `markdown.yml:40-41`
- **Confidence:** Low
- **Sink class:** silent-skip / check-shows-green-but-verifies-nothing (masked-failure).
- The `DavidAnson/markdownlint-cli2-action` step is invoked with **no `with: globs:` input and no positional globs**. In that mode the action lints exactly the file set named by the `globs` property of `.markdownlint-cli2.jsonc`. If that key is absent, empty, or scoped to a path that no longer matches the repo's markdown layout, the action can find nothing to lint and exit 0 — a green check that has stopped catching markdown defects.
- **Why this is now a finding and not a caveat:** the first pass wrote "I could not read it, so I neither confirm nor rule out." I attempted to read it: `find` over `/Users/lukas/code/claude-scanner/elixir` and a `**/.markdownlint-cli2.jsonc` glob both return nothing — only `.github/` exists in the extracted tree. The file the workflow's coverage guarantee depends on is not available to this review at all, so the guarantee is unverifiable from anything in scope. Per the re-review rule (verify by reading the neighbour, else upgrade), the neighbour cannot be read, so the assumption is upgraded.
- **Caveat on confidence:** kept Low because the `paths:` filter lists `.markdownlint-cli2.jsonc` (line 14), which strongly implies the file exists in the real repo and, by convention, carries a `globs` key — so in production this most likely works. The finding is that *this workflow does not make its own coverage self-evident*; it delegates the entire "what gets linted" contract to an out-of-tree file.
- **Recommendation:** confirm `.markdownlint-cli2.jsonc` defines a `globs` covering `**/*.md`, or add an explicit `with: globs:` so coverage is visible in the workflow itself.

## Risks / caveats

- I did not execute the workflow (no GitHub Actions runner and no full repo checkout are available here); the finding above is static (`asserted`). The action's exact empty-glob-set behavior (hard error vs. silent exit 0) is version-dependent and pinned to a SHA I did not exercise — another reason the finding is Low rather than firm.

### Ruled out

- **`strategy: { fail-fast: false }` on the `lint` job (`markdown.yml:32-33`)** — `fail-fast` affects only matrix builds, and this job declares no `matrix:`. It is a no-op, not a check that fails to fire: one leg, runs unconditionally, no masked failure. Read directly from the file. **Axes examined: matrix presence (none), leg count (one) | Not examined: none** — this is a single-implementation static YAML fact with no second implementation to differential-test against; there is no input axis to push.
- **Trigger / path-filter coverage** — `push` (branch `main`) and `pull_request` both apply the shared `*paths-filter` anchor (lines 11-17), which includes `**/*.md`, this workflow file, and the linter config. `**/*.md` matches root-level and nested markdown in GitHub path filters; `workflow_dispatch` allows manual runs. No file the workflow exists to check is excluded by the filter. Verified by reading the anchor/alias pair directly. **Axes examined: anchor/alias expansion, glob nesting semantics, all three triggers | Not examined: none.**
- **Permissions** — `permissions: { contents: read }` (lines 21-22) is least-privilege; checkout sets `persist-credentials: false` (line 38); no write scope, no secret handling, no `pull_request_target`. Nothing here is escalatable by an attacker-authored PR. Read directly. **Axes examined: permission scope, credential persistence, trigger type | Not examined: none.**
- **Action pinning** — both `actions/checkout` (line 36) and `markdownlint-cli2-action` (line 41) are pinned to full 40-hex commit SHAs, not floating tags. No supply-chain drift. Read directly. **Axes examined: pin format on both `uses:` | Not examined: none.**

### Did I verify this?

- **Verified by reading the file directly:** YAML structure, the `*paths-filter` anchor/alias pair, `fail-fast` with no matrix, `permissions`, `persist-credentials: false`, both SHA pins. (`markdown.yml`, all lines.)
- **Verified by filesystem search (new this pass):** `.markdownlint-cli2.jsonc` is **absent** from the reviewed tree — `find /Users/lukas/code/claude-scanner/elixir -name ".markdownlint*"` and glob `**/.markdownlint-cli2.jsonc` both returned nothing; only `.github/` was extracted. This is what converted the first pass's hedge into the Low finding above.
- **Not verified (cannot be, from this scope):** the actual contents of `.markdownlint-cli2.jsonc` and therefore the real linted file set; the pinned action's runtime behavior on an empty glob set. These are the residual reasons the finding is Low, not High.

### Dropped in re-review

None. The first pass asserted no findings; the sole hedged caveat has been upgraded (not dropped) to the Low-confidence finding above. Every Ruled-out entry is preserved.
