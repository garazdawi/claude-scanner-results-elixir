# /Users/lukas/code/claude-scanner/elixir/.github/workflows/release.yml
<!-- sha256: 0362a2e6995ebe354250dcc396e59954cac5bbdd82c1f0404d1ef211d8ee08c8 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/release.yml
<!-- sha256: 0362a2e6995ebe354250dcc396e59954cac5bbdd82c1f0404d1ef211d8ee08c8 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28T08:01:26Z_

# Review: `release.yml`

## Summary
Two confirmed defects carry over: Windows code-signing is silently skipped (and unsigned binaries still flow into the attested release) whenever a repo variable is unset, and a bash suffix-strip that never strips, leaving a malformed Fastly surrogate key. Re-review of the three first-pass hedges: hedge 1 (composite emits both `.zip` and `.exe`) is now **verified** by reading `release_pre_built/action.yml` and dropped as a caveat; hedge 2 splits — the `.sigstore` basename half is verified from this file, but the `bom.*` upload-glob half depends on an unreadable third-party action's output basenames and is **upgraded to a low-confidence finding**; hedge 3 is a severity-intent caveat and stays attached to the signing finding.

## Bugfixes

#### `CSCAN-release-unsigned-windows-binaries-released`
- **Finding ID:** `CSCAN-release-unsigned-windows-binaries-released`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `release.yml:146-165` (job `sign`, steps "Log in to Azure" / "Sign files with Trusted Signing"), interacting with `release.yml:167-188` (hash + upload) and the `sbom`/`upload-release` jobs.
- **Class:** Conditions that never fire, or always fire (a signing "check" that is silently skippable).
- **Issue:** Both the Azure login and the actual code-signing step are gated on `if: ${{ matrix.flavor == 'windows' && vars.AZURE_TRUSTED_SIGNING_ACCOUNT_NAME }}`. If that repo variable is empty or unset, GitHub's `&&` returns the empty string (falsy) and **both steps are skipped**. Nothing downstream re-checks that signing occurred: "Create Release Hashes" (`if: matrix.flavor == 'windows'`, line 168) still runs and hashes the *unsigned* `.exe`, the file is still uploaded as `sign-windows-elixir-otp-*` (line 187), the `sbom` job (needs `[build, sign]`) still SBOM-attests it, and `upload-release` still publishes it to the GitHub Release. There is no distinction between a fork build (where skipping is intended) and the production tag/`main` release path — the only repository-scoped guard in the file is on the unrelated "Flush cache" step (`release.yml:417`).
- **Impact:** If the signing variable is ever unset, renamed, or lost on the canonical repo, a release of **unsigned** Windows executables proceeds with zero failure signal, and the SBOM attestation then vouches for those unsigned bytes — exactly the "attestation covers unverified artifact" failure the pipeline exists to prevent. The check that is supposed to guarantee provenance of the Windows binaries is the one that silently stops running; the escaping defect class is unsigned/tampered `.exe` in an attested release.
- **Evidence:** asserted (reasoned about GitHub Actions `&&` short-circuit semantics and the absence of any downstream signing assertion; not executed in a live Actions runner — the `&&`→empty-string-falsy behaviour is not reproducible in the local Erlang/OTP sandbox, so this axis stays asserted).
- **Severity-intent caveat (carried from first pass):** the Medium rating assumes a release from the canonical repo must never silently ship unsigned Windows artifacts; under a "forks may skip signing" reading the production-path exposure is what drives severity, and I retain Medium on that basis.
- **Suggested fix:** Fail loudly when signing is expected but unavailable — e.g. add a step on the release path (tag/`main`) that errors if `matrix.flavor == 'windows' && !vars.AZURE_TRUSTED_SIGNING_ACCOUNT_NAME`, or gate the skip on `github.repository != 'elixir-lang/elixir'` so the canonical repo can never publish unsigned Windows artifacts.

#### `CSCAN-release-surrogate-key-suffix-not-stripped`
- **Finding ID:** `CSCAN-release-surrogate-key-suffix-not-stripped`
- **Severity:** Low
- **Confidence:** high
- **Location:** `release.yml:351` (job `upload-builds-hex-pm`, step "Upload Precompiled to S3"): `surrogate_key=${dest/.zip$/}`
- **Class:** Caching correctness (surrogate/purge key computed wrong).
- **Issue:** The intent is to strip the trailing `.zip` from `dest` to form the Fastly surrogate key. In a bash `${var/pattern/replacement}` expansion the pattern is a glob, so `$` is a **literal dollar sign**, not an end-of-string anchor. `dest` never contains a `$`, so no replacement happens and `surrogate_key` keeps the `.zip` suffix. The correct form is `${dest%.zip}`.
- **Impact:** The S3 object's `surrogate-key` metadata and the entry written to `purge_keys.txt` become `builds/elixir/<ref>-otp-NN.zip` instead of `builds/elixir/<ref>-otp-NN`. Because the stored surrogate key and the purge key are computed from the same wrong value they remain mutually consistent, so the in-pipeline purge still matches — the practical blast radius is limited to any *external* Fastly VCL/config that expects the extension-less key (which cannot be seen from this file), and to the inconsistency with the extension-less keys used for docs (`release.yml:376`) and the "oldest" alias (`release.yml:362`).
- **Evidence:** verified — ran `zip="elixir-otp-27.zip"; GITHUB_REF_NAME="v1.17.0"; dest=${zip/elixir/${GITHUB_REF_NAME}}; surrogate_key=${dest/.zip$/}` → `surrogate_key=v1.17.0-otp-27.zip` (unchanged), whereas `${dest%.zip}` → `v1.17.0-otp-27`.
- **Suggested fix:** Replace `${dest/.zip$/}` with `${dest%.zip}`.

#### `CSCAN-release-sbom-bom-glob-may-be-empty` — Upgraded from hedged assumption in first pass
- **Finding ID:** `CSCAN-release-sbom-bom-glob-may-be-empty`
- **Severity:** Low
- **Confidence:** low
- **Location:** `release.yml:314` (job `upload-release`, step "Upload Pre-build"): the `bom.*` argument to `gh release upload`, depending on the `SBoM` artifact assembled at `release.yml:260-268` from `steps.ort.outputs.results-sbom-*-path`.
- **Class:** Artifact name / `pattern:` glob mismatch between producer and consumer → silently empty (partial) release.
- **Issue:** `upload-release` downloads the `SBoM` artifact (merge-multiple, into the workdir root) and then uploads `bom.*`. The four SBOM files are produced at whatever paths `oss-review-toolkit/ort-ci-github-action` sets for its `results-sbom-cyclonedx-xml-path` / `-json-path` / `results-sbom-spdx-yml-path` / `-json-path` outputs. `ort/action.yml` merely re-exports those as passthrough outputs (lines 26-37, `value: "${{ steps.ort.outputs.results-sbom-... }}"`); it does not set the basenames itself. Whether those basenames actually begin with `bom.` (matching the `bom.*` glob) is determined inside the SHA-pinned third-party action `oss-review-toolkit/ort-ci-github-action@086d928…`, which is not present in this repository and cannot be read. If any of the four SBOM files is not named `bom.*` (e.g. the SPDX reporter commonly emits `*.spdx.json` / `*.spdx.yml`), that file is silently omitted from the GitHub Release while the job still succeeds.
- **Impact:** A release could be published missing one or more machine-readable SBOM files with no failure signal — a partial/silently-incomplete release rather than a security breach. The corresponding `.sigstore` attestation files are handled separately (see Ruled out — that half is verified) so provenance is not affected; only the plaintext SBOM attachments are at risk.
- **Evidence:** asserted — the basenames are set by an unreadable third-party action; I could not execute the ORT reporter to observe the emitted filenames, and reading `ort/action.yml` does not reveal them (they are passthrough outputs). Flagged low-confidence rather than dismissed because the producer side is unverifiable from the repository.
- **Suggested fix:** Pin the upload to the artifact contents explicitly (upload each `${{ steps.ort.outputs.results-sbom-*-path }}` basename, or a directory) rather than a speculative `bom.*` glob; or add a guard that fails the step if `bom.*` expands to zero matches (`shopt -s failglob` / an explicit existence check).

## Risks / caveats
- The unsigned-Windows-binaries severity depends on operational intent (fork tolerance vs. production guarantee); rated Medium on the assumption that a release from the canonical repo must never silently ship unsigned Windows artifacts. (Carried onto the finding above; not independently reproducible in the sandbox.)

### Ruled out
- **Composite action does not emit both `.zip` and `.exe` (first-pass hedge 1 — now verified).** `release_pre_built/action.yml` builds `elixir-otp-${INPUT_OTP}.zip` (`mv Precompiled.zip …`, line 30) and `elixir-otp-${INPUT_OTP}.exe` (`mv …/tmp/elixir-otp-${INPUT_OTP}.exe .`, line 47) on the Ubuntu runner. Both basenames match `build`-job upload names (`release.yml:106,112`), the `sign`-job `RELEASE_FILE` (`release.yml:132`), and every downstream download/upload glob. No artifact-name/glob mismatch. `Axes examined: producer basenames (read release_pre_built/action.yml), consumer globs (read release.yml). Not examined: none.`
- **`.sigstore` basenames do not land at the download root for the `upload-release` `elixir-otp-*.{zip,exe}.sigstore` globs (first-pass hedge 2, sigstore half — verified from this file).** The `sbom` job creates them as `attestations/$(basename "$FILE").sigstore` where `$FILE` iterates `/tmp/build-artifacts/{elixir-otp-*.*,Docs.zip}` (`release.yml:245-247`), yielding `elixir-otp-NN.zip.sigstore`, `elixir-otp-NN.exe.sigstore`, `Docs.zip.sigstore`; these are published as the `Attestations` artifact (`release.yml:270-274`) and downloaded merge-multiple by `upload-release`, so lines 307/310/313 match. This half needed no third-party knowledge. `Axes examined: sbom-job producer naming, upload-release consumer globs (both in release.yml). Not examined: none.` (The `bom.*` half is not ruled out — see finding above.)
- **`Check if variables are set up` (`release.yml:330-334`) says "skipping" but runs `exit 1`.** Not a silent failure — the mismatched message is cosmetic; the step fails the job loudly when `HEX_AWS_REGION` is unset, which is the safe direction. `Axes examined: control-flow (exit code). Not examined: none.`
- **`shell: pwsh` `Out-File` encoding for the Windows hash files (`release.yml:169-175`).** On `windows-2022`, `shell: pwsh` is PowerShell 7, whose `Out-File` default is UTF-8 (no BOM), so the `.sha1sum`/`.sha256sum` files are readable by the later `cut`/`shasum` bash steps; no UTF-16/BOM corruption. `Axes examined: pwsh-version default encoding (documented). Not examined: live-runner byte output (no Windows/pwsh in sandbox).`
- **Direct `${{ github.sha }}` interpolation inside `run:` (`release.yml:401`, `405`).** `github.sha` is a 40-char hex commit id with no attacker-controllable characters, so it is not a shell-injection vector; the untrusted values in these steps are correctly passed via `env:` and quoted `"$VAR"`. `Axes examined: value character-set of github.sha. Not examined: none.`
- **Tab character inside the ORT `version` expression (`release.yml:225`, `github.ref_type<TAB>== 'tag'`).** The value is a double-quoted YAML scalar and the `${{ }}` expression tokenizer treats the tab as ordinary whitespace, so it parses and evaluates normally. `Axes examined: YAML scalar tokenization. Not examined: none.`
- **`if: github.repository == 'elixir-lang/elixir'` on "Flush cache" (`release.yml:417`).** Correct fork guard (forks skip Fastly purge), fires as intended on the canonical repo; not a stuck/always-true condition. `Axes examined: condition polarity. Not examined: none.`
- **`fail-fast: true` / `matrix.build_docs` conditions and the `sign` OTP×flavor matrix.** Each `if: matrix.build_docs` leg (`release.yml:97,116`) fires only for the OTP-28 row that sets it (line 79), and the 6-leg sign matrix (27/28/29 × windows/linux) has a corresponding `build-*` artifact for every leg; no leg tests nothing and none is excluded. Producer legs confirmed against `release_pre_built` (single composite emits both flavors per OTP row). `Axes examined: matrix expansion, per-leg artifact presence. Not examined: none.`

## Did I verify this
- **Read** (this pass): `release.yml`, `release_pre_built/action.yml`, `ort/action.yml`.
- **Verified by reading neighbours:** hedge 1 (both `.zip`/`.exe` emitted) — cleared. `.sigstore` naming half of hedge 2 — cleared from `release.yml` itself.
- **Could not verify (upgraded to low-confidence finding):** `bom.*` upload-glob half of hedge 2 — the SBOM output basenames are set inside `oss-review-toolkit/ort-ci-github-action@086d928…`, which is not in the repository; `ort/action.yml` only passes the outputs through, so a re-read cannot resolve the basenames. Raised as `CSCAN-release-sbom-bom-glob-may-be-empty` (Confidence: low) rather than left as a silent assumption.
- **Executed:** the `${dest/.zip$/}` vs `${dest%.zip}` bash expansion (surrogate-key finding), confirming the no-op replacement. The two `asserted` items (Actions `&&` short-circuit; pwsh `Out-File` byte output) require a live GitHub-hosted runner unavailable in the Erlang/OTP sandbox and remain asserted; I did not upgrade or clear their untried axes.

### Dropped in re-review
- **First-pass caveat "composite action artifact presence is assumed, not verified."** Dropped as a caveat because it is now verified: `release_pre_built/action.yml:30,47` emit exactly `elixir-otp-${otp}.zip` and `elixir-otp-${otp}.exe`. Moved to Ruled out with `Not examined: none`.
- No **findings** were dropped. Both first-pass findings are retained unchanged.
