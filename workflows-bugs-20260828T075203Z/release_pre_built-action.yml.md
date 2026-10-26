# /Users/lukas/code/claude-scanner/elixir/.github/workflows/release_pre_built/action.yml
<!-- sha256: c5cc0381f0cb780bf88cdbae28cfc14eb4c686e4a22a98bbd2c3fd607d4fc80d -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/release_pre_built/action.yml
<!-- sha256: c5cc0381f0cb780bf88cdbae28cfc14eb4c686e4a22a98bbd2c3fd607d4fc80d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

# Release Pre-build Composite Action — Bugfix Review (re-review)

## Summary

The action is structurally sound (all `run:` steps pin `shell: bash`, both `uses:` are SHA-pinned, no triggers/matrix/cache to misconfigure), but its documented `build_docs` toggle uses a string-truthiness gate that treats a literal `false` as "build docs." The re-review resolved the two hedges flagged in the first pass: (1) the caller (`release.yml`) was read and confirmed to use the sentinel-string convention, so the `build_docs` bug is latent-but-real in the public input surface; (2) the `Get ExDoc ref` robustness caveat was **executed** in the sandbox — under GitHub Actions' default `bash -eo pipefail`, every failure mode aborts loudly (either at the `Get ExDoc ref` step or at the subsequent `actions/checkout`), so it stays ruled out with run evidence rather than reasoned assertion.

## Bugfixes

#### `CSCAN-action-build-docs-string-truthy`
- **Finding ID:** `CSCAN-action-build-docs-string-truthy`
- **Severity:** Medium
- **Confidence:** medium
- **Location:** `release_pre_built/action.yml:52,62,69,77` — steps `Get ExDoc ref`, `actions/checkout`, `Build ex_doc`, `Build Docs`; clause `if: ${{ inputs.build_docs }}`
- **Class:** Conditions that never fire, or always fire
- **Issue:** The four doc-building steps are gated on `if: ${{ inputs.build_docs }}`. Composite-action inputs are always strings, and the GitHub expression evaluator's boolean coercion of a string is "non-empty ⇒ true" — it does **not** special-case the word `false`. So passing `build_docs: false` (a YAML boolean, serialized to the string `"false"`) makes the condition truthy and the docs are built anyway. Only the empty string (input omitted) is falsy. The documented input contract — *"Whether docs have to be built"* — invites exactly the boolean `true`/`false` call pattern that this gate mishandles.
- **Impact:** A caller that follows the natural, documented contract and passes `build_docs: false` gets the ex_doc checkout, compile, and `make Docs.zip` steps run regardless — wasted CI minutes, an unexpected network checkout of `elixir-lang/ex_doc`, and doc artifacts produced for a leg that asked for none. The failure is silent: nothing errors, the toggle simply does not toggle. The class of defect that escapes is "feature flag that cannot turn off" — the off state is unreachable via the obvious boolean value.
- **Evidence:** Caller verified by reading `release.yml:66-119`. The build matrix (`release.yml:72-83`) sets `build_docs: build_docs` **only** on the OTP-28 row; OTP-27 and OTP-29 rows omit the key. The step passes it through as `build_docs: ${{ matrix.build_docs }}` (`release.yml:94`). So the sole current caller relies on the sentinel-string convention (`"build_docs"` for on, unset→empty string for off), never `true`/`false` — the bug is **latent today but lives in the public input surface** of the action. The truthiness half of the claim (a non-empty string, including `"false"`, is truthy in `${{ }}` inside `if:`) rests on documented GitHub expression-coercion semantics; the GitHub expression evaluator is not runnable in this sandbox, which is why this remains a medium-confidence finding rather than a run-verified one.
- **Suggested fix:** Gate on an explicit comparison, e.g. `if: ${{ inputs.build_docs == 'true' }}` (and have the caller pass `'true'`), or document/default the sentinel-string convention the current caller already relies on and give the input a `default: ''`.

## Risks / caveats

- I reviewed this file and its sole caller (`release.yml`). Whether the latent `build_docs` bug ever fires depends on callers; the one caller inspected is safe today (sentinel-string convention), but any other/future caller using boolean values is not.
- The truthiness finding rests on documented GitHub expression semantics rather than an observed Actions run (the evaluator cannot be exercised in this sandbox); the jq-robustness question below *was* run.

### Ruled out

- **`Get ExDoc ref` yields a silent-bad ref (jq/curl error handling)** — line 58, `ref=v$(curl -s https://hex.pm/api/packages/ex_doc | jq --raw-output '.latest_stable_version')`.
  - **Verdict:** ruled out — fails **loudly** in every failure mode; no silent-corruption path exists.
  - **Executed evidence:** Under GitHub Actions' default shell for `shell: bash` (`bash --noprofile --norc -eo pipefail {0}`), I ran each axis:
    - *curl network/DNS failure* → the `curl | jq` pipe fails; `pipefail` propagates, and the `ref=v$(...)` assignment aborts the step (observed `step_exit=7`; "REACHED_END" never printed).
    - *non-JSON response (e.g. Fastly/HTML error page)* → `jq` parse error, exit 5; assignment aborts the step (observed `step_exit=5`).
    - *valid JSON, key missing* → `jq --raw-output` prints the literal `null` (exit 0) → `ref=vnull`; step succeeds but `actions/checkout` (line 61, `ref: ${{ env.EX_DOC_REF }}`) then fails on the nonexistent ref `vnull`.
    - *valid JSON, key value `null`* → same as above, `ref=vnull` → checkout fails loudly.
    - *empty jq output (exit 0)* → `ref=v` → checkout fails loudly on nonexistent ref `v`.
  - **Axes examined:** curl failure (pipefail abort), malformed-JSON parse error (jq exit), missing key (`null` literal), null-valued key, empty output → downstream checkout. **Not examined:** none.
- **Shell assumptions / PowerShell fallthrough** — every one of the six `run:` steps (lines 27, 36, 42, 53, 70, 78) explicitly sets `shell: bash`; each was read. No step is left to a default shell. **Axes examined:** all six `run:` steps. **Not examined:** none.
- **Unpinned actions (supply-chain)** — both `uses:` entries are commit-SHA-pinned: `erlef/setup-beam@5304e04…` (line 21) and `actions/checkout@11bd719…` (line 61). No tag/branch-pinned action exists in the file. **Axes examined:** both `uses:` refs. **Not examined:** none.
- **`continue-on-error` / swallowed exits** — the string `continue-on-error` does not appear in the file, and no `run:` body ends in `|| true` or pipes a gating command into a masker. Verified by full read. **Axes examined:** whole-file grep + each `run:` body. **Not examined:** none.
- **Matrix / trigger / concurrency / cache errors** — not applicable by construction: the file is `runs.using: composite` with no `on:`, `jobs:`, `strategy:`, `concurrency:`, or `actions/cache` present. Verified structurally against the whole file. **Axes examined:** file structure. **Not examined:** none.
- **`persist-credentials` on the ex_doc checkout** — line 67 sets `persist-credentials: false`; the credential-persistence risk is closed. **Axes examined:** the one code-handling checkout in the file. **Not examined:** none.
- **Untrusted-input script injection** — no `${{ github.event.* }}` / PR-title / branch-name interpolation appears inside any `run:` body; the only interpolated inputs (`inputs.otp`, `inputs.otp_version`) are passed via `env:` (lines 32-33, 48-50) and referenced as `"$VAR"`. This is a `push`/`dispatch`-path composite action (called from `release.yml`, not a `pull_request` trigger). **Axes examined:** each `run:` body for direct `${{ }}` use. **Not examined:** none.
- **Unconditional NSIS / Windows-installer steps (lines 35–50)** — these correctly run on every matrix leg by design (each OTP build ships a Windows installer); absence of an `if:` here is intended, not a defect. **Axes examined:** step guards vs. caller matrix. **Not examined:** none.

### Did I verify this?

- **`build_docs` string-truthiness finding** — caller path **verified** by reading `release.yml` (matrix at 72-83, pass-through at 94): the sole caller uses the sentinel-string convention, confirming the bug is latent-but-present in the input surface. The truthiness coercion itself is **documented-semantics, not run** — the GitHub expression evaluator cannot be exercised in the sandbox — hence medium confidence, retained as a finding (not upgraded, not dropped).
- **`Get ExDoc ref` robustness** — **verified by execution.** First pass hedged this as "reasoned, not run"; I ran `jq`/`curl` and the `ref=v$(...)` assignment under `bash -eo pipefail` across all five failure axes. Every path fails loudly (step abort via pipefail/jq-exit, or downstream `actions/checkout` on a bad ref). Now ruled out with `Not examined: none`.
- No files were modified.

### Dropped in re-review

None. Both prior conclusions are preserved: the `build_docs` finding is retained (caller now verified), and the jq caveat remains ruled out — now on executed evidence rather than reasoning.
