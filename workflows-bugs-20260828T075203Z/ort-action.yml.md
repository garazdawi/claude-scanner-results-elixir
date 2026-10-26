# /Users/lukas/code/claude-scanner/elixir/.github/workflows/ort/action.yml
<!-- sha256: 017506d56c331e61801fe109468e472ab535f3ab1cd6b4d2dc79f038a6c9d330 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/ort/action.yml
<!-- sha256: 017506d56c331e61801fe109468e472ab535f3ab1cd6b4d2dc79f038a6c9d330 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

# OSS Review Toolkit composite action review — `ort/action.yml`

## Summary
The action's ScanCode cache key can never invalidate, and `sed`-based version substitution silently corrupts SBOM metadata when the version input contains a `&`. Re-review resolved the caller side of the `version` input (it comes from `release.yml:225` = tag `v*` name or `github.sha`, and `license_compliance.yml:41` = `github.sha`), which bounds the real-world likelihood of the `sed` finding, and upgraded two prior "could not verify" caveats to low-confidence findings because their unknowns live in an external pinned action / the ORT binary, not in any callgraph neighbour I can read.

## Bugfixes

#### `CSCAN-action-scancode-cache-key-never-invalidates`
- **Finding ID:** `CSCAN-action-scancode-cache-key-never-invalidates`
- **Severity:** Medium
- **Confidence:** high (mechanism); impact is contingent — see cross-reference to `CSCAN-action-provenant-scanner-unverified` below
- **Location:** `ort/action.yml:84-88`, step `Cache ScanCode` (`actions/cache`)
- **Class:** Caching correctness
- **Issue:** The cache key is `${{ runner.os }}-scancode` (line 88) — it contains no ScanCode/ORT version, no image digest, and no content hash. `actions/cache` treats an exact key hit as immutable: on a hit it restores and does **not** re-save. Once the key is populated on a given runner OS, that cached `~/.cache/scancode-tk` payload (line 87) is frozen forever, even when the ORT image (`ghcr.io/oss-review-toolkit/ort:92.2.0`, line 94) — and the ScanCode license index it ships — is bumped.
- **Impact:** After the first successful run the license/scan cache never refreshes. When ScanCode or its rule/license database changes with a new ORT image, the scanner runs against a stale cached index and can silently emit outdated license findings and SBOM contents — a wrong-but-green result. Because this SBOM feeds the signed/attested release path (`release.yml` `sbom` job at 220-237 consumes this action's outputs and `actions/attest-sbom` attests them), stale output is attested as current.
- **Cross-reference:** the concrete impact assumes the configured scanner actually reads `~/.cache/scancode-tk`. The action configures `--scanners Provenant` (line 111), not ScanCode; if that scanner is not ScanCode-backed the cache is never read/written and the *staleness* impact is moot (though the key is still misdesigned). See `CSCAN-action-provenant-scanner-unverified`.
- **Evidence:** asserted (documented `actions/cache` semantics: exact-key hit ⇒ restore-only, no save; key here has no version/content component). Not executed — no runner/cache backend available in the sandbox; this is a documented-behaviour claim, not a reproduced one.
- **Suggested fix:** Fold the ORT image tag (or a scancode version/digest) into the key, e.g. `key: ${{ runner.os }}-scancode-ort-92.2.0`, so an image bump forces a fresh cache; optionally add a `restore-keys:` prefix for warm starts.

#### `CSCAN-action-sed-version-ampersand-corruption`
- **Finding ID:** `CSCAN-action-sed-version-ampersand-corruption`
- **Severity:** Medium
- **Confidence:** high (corruption reproduced); low real-world likelihood (see caller analysis)
- **Location:** `ort/action.yml:71-79`, step `Setup ORT Config` (the `sed` calls at lines 72, 77, 78, 79 interpolating `${ELIXIR_VERSION}`)
- **Class:** Shell/`sed` metacharacter injection → silent data corruption
- **Issue:** `inputs.version` (bound to `ELIXIR_VERSION`, line 81) is interpolated unescaped into `sed` replacement text. In a `sed` replacement `&` expands to the whole matched string and `\` is an escape. A version value containing `&` does not error — it produces the wrong string. The `/`-delimited substitutions (lines 72/77/78) and the `|`-delimited one (line 79) fail *loudly* on a delimiter collision, and because a composite `shell: bash` step runs under `-eo pipefail`, those loud failures abort the step (safe-ish). The `&` case is the silent one.
- **Caller analysis (new this pass):** `version` is supplied by exactly two callers — `release.yml:225` (`github.ref_type == 'tag' && github.ref_name || github.sha`) and `license_compliance.yml:41` (`github.sha`). A `github.sha` is `[0-9a-f]{40}` (no metacharacters). A tag name reaches here only via `release.yml`'s `on.push.tags: [v*]` filter, so it is a maintainer-created `v…` tag. Normal tags (`v1.18.0`, `v1.18.0-rc.0`) contain only `.`/`-`, which are literal in a `sed` *replacement* and reproduce correctly (verified below). A `&` therefore requires a maintainer to push a tag literally containing `&` — an operator-controllable, not attacker-controllable, path. This bounds likelihood to "low" but does not eliminate the silent-corruption class.
- **Impact:** For a version input containing `&`, the SPDX `versionInfo` (line 77), the `.ort.yml` package coordinates (line 78), the repo URL rewrite (line 79), and the per-component package-configuration coordinates (line 72) are written with corrupted version strings, with no error and a green build; the SBOM/attestation then vouches for a wrong version identifier.
- **Evidence:** verified — executed in the sandbox:
  ```
  ver=[a&b]           out=[versionInfo: 'a# elixir-version-insertb']   (& pulled in the matched pattern — silent)
  ver=[a/b]           out=[# elixir-version-insert]  err=[sed: ... unknown option to `s'] (loud, file unchanged)
  ver=[v1.18.0-rc.0]  out=[versionInfo: 'v1.18.0-rc.0']  (normal tag: correct)
  ver=[0a1b2c3d4e5f6071] out=[versionInfo: '0a1b2c3d4e5f6071']  (SHA: correct)
  ```
  The `|`-delimited line-79 form was likewise observed to fail loudly on a `|` in the version and to mis-expand on `&` (`https://x/y@a<matched>b`).
- **Suggested fix:** Escape the value before substitution (escape `&`, `\`, and the delimiter), or avoid `sed` regex — write the version via `awk`/`envsubst`/a templating step with fixed placeholders rather than into a regex replacement.

#### `CSCAN-action-provenant-scanner-unverified` — *Upgraded from hedged assumption in first pass*
- **Finding ID:** `CSCAN-action-provenant-scanner-unverified`
- **Severity:** Low
- **Confidence:** low
- **Location:** `ort/action.yml:111` (`--scanners Provenant`), interacting with `ort/action.yml:84-88` (ScanCode cache)
- **Class:** Config correctness (tool/argument mismatch → silent under-scanning or dead cache)
- **Issue:** The `ort-cli-scan-args` pass `--scanners Provenant` (line 111). "Provenant" is not a scanner name I can confirm against ORT's registry, and it is not obviously ScanCode. If it is a typo or an unregistered scanner, ORT may scan with a different/empty scanner set than intended (silent under-scanning of license/provenance data that flows into the attested SBOM). If it is real but not ScanCode-backed, the `Cache ScanCode` step (path `~/.cache/scancode-tk`) is never read or written and is dead weight (and the staleness impact of `CSCAN-action-scancode-cache-key-never-invalidates` is moot).
- **Why not verified:** the scanner registry lives inside the pinned `oss-review-toolkit/ort:92.2.0` image / the `ort-ci-github-action` — neither is a callgraph neighbour present in this repo, so reading neighbour modules cannot adjudicate the name. I could not execute ORT (no image available in the sandbox). Per the re-review rule I therefore upgraded the prior caveat to a low-confidence finding rather than leaving it as an unverifiable aside.
- **Suggested fix:** Confirm `Provenant` is a valid ORT scanner id for image `92.2.0` (and whether it is ScanCode-backed); if it is not the intended scanner, correct the name and either keep the ScanCode cache (with a version-scoped key) or remove it as dead weight.

#### `CSCAN-action-ort-run-list-trailing-empty-element` — *Upgraded from hedged assumption in first pass*
- **Finding ID:** `CSCAN-action-ort-run-list-trailing-empty-element`
- **Severity:** Low
- **Confidence:** low
- **Location:** `ort/action.yml:95-104`, the folded `run:` list; specifically line 104
- **Class:** Config correctness (list construction)
- **Issue:** When `inputs.upload-reports != 'true'`, line 104 evaluates to `''`, so the folded (`>-`) list becomes `labels, cache-dependencies, cache-scan-results, analyzer, scanner, advisor, evaluator, reporter, ` — i.e. it ends with `reporter,` followed by an empty final element. Whether `ort-ci-github-action@086d928` tolerates a trailing/empty comma-separated entry or errors/mis-parses it is unknown.
- **Why not verified:** the parsing lives in the external pinned action `oss-review-toolkit/ort-ci-github-action` — not a neighbour module in this repo — and I could not execute the action. Upgraded from caveat to low-confidence finding accordingly.
- **Suggested fix:** Emit the whole `upload-results` token conditionally without a dangling separator, e.g. build the list so the conditional element carries its own leading comma only when present (`${{ inputs.upload-reports == 'true' && ', upload-results' || '' }}` appended after `reporter`), or filter empties before passing to the action.

## Risks / caveats
- `default: false` (YAML boolean) on `fail-on-violation`/`upload-reports` (lines 13, 17) compared against the string `'true'` (lines 104-105) is correct: action inputs coerce to strings, so the defaults evaluate to no-upload / no-fail as intended — not a finding.
- Cross-file invariant confirmed this pass: only `release.yml` sets `fail-on-violation: true`? — no. `license_compliance.yml:39` sets `fail-on-violation: true`; `release.yml` does **not** pass it, so the release SBOM path uses the `false` default and does not fail on license violations. This matches the standing threat model and is a deliberate design choice, not a finding in this file.

### Ruled out
- **`if:` conditions that never/always fire** — composite action with no `if:` on any step; every step runs unconditionally. Verified by re-reading all steps (lines 42-112); none carry an `if:`. Axes examined: presence of `if:` on each step. Not examined: none (structural fact from the file).
- **`continue-on-error` / swallowed failures** — no step sets `continue-on-error`; the single `run:` block (lines 54-79) uses no `|| true` and no exit-masking pipe. Composite `shell: bash` runs under `-eo pipefail`, so the `sed` delimiter failures propagate and abort the step. Verified by re-reading the `run:` block. Axes examined: exit-status masking constructs, shell flags. Not examined: none.
- **Matrix / trigger / concurrency errors** — not applicable: a composite action defines no `on:`, no `strategy.matrix`, and no `concurrency`. Verified structurally (`runs.using: composite`, line 40). Axes examined: presence of trigger/matrix/concurrency keys. Not examined: none.
- **Unpinned actions** — all three `uses:` are SHA-pinned: `actions/checkout@11bd719` (44), `actions/cache@d4323d4` (85), `oss-review-toolkit/ort-ci-github-action@086d928` (92). Verified by re-reading each `uses:`. Axes examined: SHA-vs-tag pinning of each `uses:`. Not examined: none.
- **Shell assumptions on Windows** — the only `run:` step declares `shell: bash` (line 53); PowerShell-quoting drift does not apply. Verified by re-reading the step. Axes examined: declared shell. Not examined: none.

### Did I verify this
- **`sed` `&` corruption:** Yes — reproduced in the sandbox with a working shell/`sed`; `a&b` yields `versionInfo: 'a# elixir-version-insertb'` (silent), `a/b` fails loudly and leaves the file unchanged, and normal tag/SHA values substitute correctly. Executed evidence cited above.
- **`version` input provenance:** Yes — read both callers (`release.yml:225`, `license_compliance.yml:41`). Confirmed values are a `v*` tag name or `github.sha`, which bounds the `&` corruption to an operator (tag-creator) path, not an attacker/PR path. This narrowed likelihood but did not clear the finding.
- **Cache-key immutability:** No execution — no `actions/cache` backend in the sandbox. Claim rests on documented `actions/cache` restore-only-on-exact-hit semantics; kept as an asserted (not reproduced) mechanism.
- **`--scanners Provenant` validity and trailing-comma parsing:** Not verified — both depend on the external pinned ORT image / `ort-ci-github-action`, which are not callgraph neighbours in this repo and could not be executed. Reading neighbour workflows (`release.yml`, `license_compliance.yml`) does not adjudicate ORT-internal behaviour, so per the re-review rule these were upgraded from caveats to low-confidence findings rather than left as unverifiable asides.

### Dropped in re-review
None. All prior findings are preserved; the two prior "Risks / caveats" items that carried "I could not verify" hedges (`--scanners Provenant`, trailing-comma) were upgraded to low-confidence findings rather than dropped.
