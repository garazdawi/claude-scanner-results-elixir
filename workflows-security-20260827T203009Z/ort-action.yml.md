# /Users/lukas/code/claude-scanner/elixir/.github/workflows/ort/action.yml
<!-- sha256: 017506d56c331e61801fe109468e472ab535f3ab1cd6b4d2dc79f038a6c9d330 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/ort/action.yml
<!-- sha256: 017506d56c331e61801fe109468e472ab535f3ab1cd6b4d2dc79f038a6c9d330 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-27_

The single flagged hedge phrase ("would warrant") lived in the ScanCode-cache caveat, resting on an unexecutable platform-isolation assumption (GitHub Actions cache branch-scoping). Reading the two caller modules cannot clear that axis — cache isolation is platform behaviour, not something the callers reveal, and I cannot execute it in the sandbox. Per "prefer downgrading to upgrading," I have upgraded the ScanCode cache-key weakness from a ruled-out hygiene note to a **low-confidence finding** (`CSCAN-action-scancode-cache-key`) rather than continue to assert the isolation. All other findings were verified by execution and stand.

## Summary

`ort/action.yml` is a well-hardened composite action (SHA-pinned actions, `persist-credentials: false`, `${{ }}` values passed via `env:`), but it trusts two mutable third-party references that feed a signed/published SBOM, contains a latent (currently non-outsider-reachable) `sed`-based command-injection on its `version` input, and keys its ScanCode cache on `runner.os` alone.

## Security

This action is invoked by two callers, both re-read this pass:
- `license_compliance.yml` (`push` / **`pull_request`** / `workflow_dispatch`; `permissions: contents: read`; no secrets; passes `version: "${{ github.sha }}"` at line 41) — the fork-PR path runs with a read-only token.
- `release.yml`'s `sbom` job (`needs: [build, sign]`; passes `version: "${{ github.ref_type == 'tag' && github.ref_name || github.sha }}"` at line 225; its downstream `attest-sbom`/`upload-artifact` steps hold `id-token: write` + `attestations: write` + `contents: write`) — the signed-release path, **not** fork-PR reachable.

Findings are rated against those two realities.

#### `CSCAN-action-ort-config-mutable-ref`
- **Finding ID** — `CSCAN-action-ort-config-mutable-ref`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `ort/action.yml:44-49`, step `Fetch Default ORT Config` (`repository: oss-review-toolkit/ort-config`, `ref: "main"`).
- **Class** — Unpinned third-party actions (mutable content ref) / artifact-poisoning.
- **Issue** — The action pins `actions/checkout` by SHA (line 44) but checks out the *content* of `oss-review-toolkit/ort-config` at the mutable branch `main` (line 47). That tree is moved to `$HOME/.ort/config` (line 58), then the in-repo `.ort/config/config.yml` is appended and `evaluator.rules.kts` overridden (lines 61-64) — but the upstream `main` tree remains the base config (curations, resolutions, package configs) that drives analyzer/scanner/advisor.
- **Impact / threat** — Whoever controls (or compromises) `ort-config`'s `main` branch can silently alter the config governing license/vulnerability findings. In the `release.yml` `sbom` job this config feeds the SBOM that is then **attested and signed** and published to everyone who installs Elixir — e.g. suppressing a real advisory or injecting a false curation so the signed SBOM misrepresents the shipped software. Trigger is upstream-repo compromise (the threat model's "tertiary" attacker), not the fork-PR outsider; the fork-PR path (`license_compliance.yml`) only taints a throwaway PR scan.
- **Evidence** — verified. Read `ref: "main"` at `ort/action.yml:47`; traced the merged config into the signed-SBOM path — `release.yml:222` invokes this action from the `sbom` job, whose outputs (lines 233-237) flow into `actions/attest-sbom@…` (line 229) with `sbom-path` set to this action's SPDX output.
- **Suggested fix** — Pin to a reviewed commit and bump deliberately:
  ```yaml
  with:
    repository: oss-review-toolkit/ort-config
    ref: <full-commit-sha>   # was "main"
    path: ".ort-config"
    persist-credentials: false
  ```

#### `CSCAN-action-ort-image-mutable-tag`
- **Finding ID** — `CSCAN-action-ort-image-mutable-tag`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `ort/action.yml:94`, step `Run OSS Review Toolkit` (`image: ghcr.io/oss-review-toolkit/ort:92.2.0`).
- **Class** — Unpinned third-party actions (mutable OCI tag).
- **Issue** — The ORT CI action itself is SHA-pinned (`@086d928…` line 92), but the container image it runs is referenced by the mutable tag `92.2.0` (line 94), not by digest. A registry tag can be re-pushed to point at different bytes.
- **Impact / threat** — If the `ort` image/tag is repushed (upstream or registry compromise), arbitrary code runs inside the scan job. In `release.yml`'s `sbom` job the downstream signing steps hold `id-token: write` + `attestations: write` + `contents: write`, so a poisoned container can forge/alter the signed SBOM and OIDC-backed attestation that ship to all downstream installers. Same tertiary-attacker trigger as above.
- **Evidence** — verified. Line 94 is `image:tag`, not `image@sha256:…`.
- **Suggested fix** — Pin by digest, keeping the version in a comment:
  ```yaml
  image: ghcr.io/oss-review-toolkit/ort@sha256:<digest>   # 92.2.0
  ```

#### `CSCAN-action-version-sed-injection`
- **Finding ID** — `CSCAN-action-version-sed-injection`
- **Severity** — Low
- **Confidence** — high
- **Location** — `ort/action.yml:71-79` (four `sed` calls) with `env: ELIXIR_VERSION: "${{ inputs.version }}"` at line 81.
- **Class** — Expression injection (into a `sed` script rather than directly into the shell).
- **Issue** — `inputs.version` is passed via `env:` (correctly avoiding *direct* shell injection), but `${ELIXIR_VERSION}` is interpolated verbatim into `sed` `s///` scripts: line 72 (`s/(\"…:)\"/\1${ELIXIR_VERSION}\"/`), line 77 (`s/# elixir-version-insert/versionInfo: '${ELIXIR_VERSION}'/`), line 78, and the `|`-delimited line 79 (`s|…|${ELIXIR_REPO}@${ELIXIR_VERSION}|`). A value containing sed delimiters plus the GNU-sed `e` flag executes an attacker-chosen command; a value with a stray `/`/`|` aborts the step under `bash -eo pipefail`.
- **Impact / threat** — If a caller ever supplies an attacker-influenced `version`, this is command execution in the ORT job (in the release path, whose downstream steps carry `id-token`/`attestations: write`). **Through the two current callers it is not outsider-reachable:** `license_compliance.yml:41` passes `github.sha` (40-char hex — no sed metacharacters), and `release.yml:225` passes `github.ref_name` only on a tag (push-access-gated, treated as out-of-band) else `github.sha`. So today this is a latent/defense-in-depth defect in a reusable, release-critical action, not a live outsider exploit.
- **Evidence** — verified (re-executed this pass). GNU sed 4.9, isolated `/tmp/ortsedtest`: with `ELIXIR_VERSION="'; touch PWNED_C; :/e ; #"` the line-77 `sed` created `PWNED_C` (`>>> PWNED_C CREATED (line-77 RCE) <<<`, with `sh: 1: versionInfo:: not found` confirming shell execution of the pattern space via the `e` flag); a bare `a/b/c` under `set -eo pipefail` returned `sed: unknown option to 's'` / `dos-exit=1` (step-failure DoS).
- **Suggested fix** — Validate `version` before use and fail closed (a commit SHA or a `vX.Y[.Z]` tag needs none of `/ | ' \ &` or newlines):
  ```bash
  case "$ELIXIR_VERSION" in
    *[!A-Za-z0-9._-]*) echo "refusing unsafe version: $ELIXIR_VERSION" >&2; exit 1 ;;
  esac
  ```

#### `CSCAN-action-scancode-cache-key`  *(Upgraded from hedged assumption in first pass)*
- **Finding ID** — `CSCAN-action-scancode-cache-key`
- **Severity** — Low
- **Confidence** — low
- **Location** — `ort/action.yml:84-88`, step `Cache ScanCode` (`path: ~/.cache/scancode-tk`, `key: ${{ runner.os }}-scancode`).
- **Class** — Cache poisoning (build-input integrity).
- **Issue** — The cache key carries only `runner.os` — no content/version/lockfile hash. Any writer to that key namespace persists a `scancode-tk` cache that a later default-branch `sbom` run (the signed-release path) will restore as a scanner input.
- **Impact / threat** — If a poisoned `~/.cache/scancode-tk` is restored into the release `sbom` job, it could alter ScanCode's license/scan results feeding the attested-and-signed SBOM. The first pass ruled this out for outsider-reachability by asserting GitHub Actions cache **branch-scoping** (fork-PR cache writes isolated to the PR merge ref, unable to populate the default-branch scope). **I cannot execute that platform-isolation behaviour in the sandbox, and reading the two caller modules does not exercise it** — so I am not willing to assert it. Rated low-confidence: if branch-scoping holds (documented behaviour), the only residual writer is a default-branch pusher (push-access, out of scope) and this is hygiene; if it were ever relaxed, the static key becomes an outsider cache-poisoning vector into the signed path.
- **Evidence** — asserted. Read `key: ${{ runner.os }}-scancode` at line 88 (no content hash). The isolation claim that would downgrade this is an unexecuted platform invariant.
- **Suggested fix** — Add a content/version dimension to the key (e.g. hash of the ORT image tag or ScanCode version) and/or a `restore-keys` policy so a stale/poisoned entry cannot silently satisfy a release scan.

### Ruled out

- **Expression injection into `run:` (step `Setup ORT Config`, lines 53-82)** — the `run:` body references only shell variables (`$ELIXIR_VERSION`, `$ELIXIR_REPO`, `$HOME`, `$FILE`, `$COMPONENT`); every `${{ }}` (`inputs.version`, `github.server_url`, `github.repository`) is confined to the `env:` block (lines 80-82). `github.repository`/`server_url` cannot contain shell/sed metacharacters, and `$COMPONENT`/`$FILE` are quoted at every use (lines 69-73). The residual `sed`-into-`ELIXIR_VERSION` weakness is reported above as `CSCAN-action-version-sed-injection`, not dismissed.
  - Axes examined: direct-shell-metachar (`env:`-confined), quoting of loop vars | Not examined: none.
- **`pull_request_target` / `workflow_run`** — verified this pass: neither caller declares them. `license_compliance.yml:6-9` uses `push` / `pull_request` / `workflow_dispatch`; `release.yml`'s `sbom` job is reached via `needs: [build, sign]` on the release triggers, not a fork-context trigger. So there is no write-token/secret-in-fork-context shape here.
  - Axes examined: caller trigger set (both callers read) | Not examined: none.
- **`secrets` exposure** — this file references no `secrets.*` and passes none to the ORT container or to `run:`. Verified `license_compliance.yml` supplies no `secrets:` to the `uses: ./.github/workflows/ort` step (lines 34-41) and runs `permissions: contents: read` (line 12) — the intended fork-PR containment holds.
  - Axes examined: this file's `secrets.*` refs, fork-PR caller's passed inputs/permissions | Not examined: none.
- **Missing `permissions:` block** — a composite action cannot declare `permissions:`; effective scope is the caller's job (`contents: read` on the PR path — least-privilege; the elevated `id-token`/`attestations`/`contents: write` is granted only by the non-fork release job). Not a defect of this file.
  - Axes examined: composite-action schema, both callers' granted scopes | Not examined: none.
- **Unpinned `uses:` actions** — `actions/checkout@11bd719…` (line 44), `actions/cache@d4323d4…` (line 85) and `oss-review-toolkit/ort-ci-github-action@086d928…` (line 92) are all full-SHA pinned with version comments. Only the *content ref* (`ort-config@main`) and the *container image* (`ort:92.2.0`) are mutable, and those are reported above.
  - Axes examined: every `uses:` line pin format | Not examined: none.

## Did I verify this

- **Verified by execution (fresh this pass):** the `version` sed-injection RCE (GNU sed 4.9, line-77 form created `PWNED_C` via the `e`-flag; shell error confirms pattern-space execution) and the stray-delimiter DoS (`exit=1` under `set -eo pipefail`).
- **Verified by reading callgraph neighbours (fresh this pass):** both callers of this action. `license_compliance.yml` (full file) — `pull_request` trigger, `contents: read`, no secrets, `version: "${{ github.sha }}"`. `release.yml` `sbom` job — `version: "${{ github.ref_type == 'tag' && github.ref_name || github.sha }}"` (line 225), outputs flowing into `actions/attest-sbom` (lines 229-237) with `id-token`/`attestations`/`contents: write` in scope. These establish input-reachability and signed-output impact for all four findings.
- **Could not verify (kept as-is, not asserted):** GitHub Actions cache branch-scoping isolation — an unexecutable platform invariant. Reading the callers does not exercise it, so rather than continue to assert it (the first pass's "would warrant" hedge), the ScanCode cache-key weakness is now a low-confidence finding (`CSCAN-action-scancode-cache-key`). Also unverified: the ORT container's runtime behaviour and whether `ort-config@main` tampering can escalate from SBOM-content tampering to in-image code execution — the two mutable-ref findings are rated on tamper-of-signed-output impact, which holds regardless.

### Dropped in re-review

None. All four prior findings are preserved; the ScanCode cache item moved *up* (from ruled-out hygiene to a low-confidence finding), not dropped.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **ScanCode cache poisoning (lines 84-88, key `${{ runner.os }}-scancode`)** — the key carries no content/version hash, but GitHub Actions caches are branch-scoped: a fork-PR run's cache writes are isolated to the PR merge ref and cannot populate the default-branch cache scope that `release.yml` restores from, so an unprivileged outsider cannot poison the signed-release scan input. The only residual writer is a default-branch pusher (push-access, out of scope). Noted as hygiene, not a live finding.
