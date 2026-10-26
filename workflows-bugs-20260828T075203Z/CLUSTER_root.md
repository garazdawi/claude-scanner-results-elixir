# Cluster cross-file findings

#### `CSCAN-cluster-release-not-gated-on-license-compliance`

- **Finding ID** — CSCAN-cluster-release-not-gated-on-license-compliance
- **Severity** — Medium
- **Confidence** — high
- **Files involved** —
  - `release.yml` (sbom job: lines 190–226, esp. the `ort` step 220–225 which omits `fail-on-violation`; job dependency graph `needs: [create_draft_release, build, sign, sbom]` at 278; hex upload job 316–321)
  - `license_compliance.yml` (lines 34–41, `fail-on-violation: true`)
  - `ort/action.yml` (input default at lines 10–13 `fail-on-violation … default: false`; the `fail-on` wiring at line 105 `"${{ inputs.fail-on-violation == 'true' && 'violations,issues' || '' }}"`)
- **Finding shape** — Inherited assumption broken / invariant set in one file and violated in another (state-machine gating inconsistency across workflows).
- **Evidence** —
  - `ort/action.yml:105` only passes `fail-on: violations,issues` to the underlying ORT action when `inputs.fail-on-violation == 'true'`; otherwise `fail-on` is empty and ORT never fails the step.
  - `license_compliance.yml:39` sets `fail-on-violation: true`, so that standalone workflow *does* red-X on a license/policy violation.
  - `release.yml:220–225` invokes the same `./.github/workflows/ort` composite **without** `fail-on-violation`, so it inherits the `default: false` from `ort/action.yml:13` — the release-path ORT run cannot fail on a violation.
  - The two workflows are independent: `release.yml`'s DAG (`needs:` at lines 123, 192, 278, 319) never depends on `license_compliance.yml`, and both fire on the same tag push (`release.yml:6–13` vs `license_compliance.yml:6–9`). Nothing in `release.yml` waits for or checks the license gate’s result, and `sbom`→`upload-release`/`upload-builds-hex-pm` proceed to attest, sign, and publish regardless.
- **What the per-file pass missed** — A single-file reviewer of `release.yml` sees an ORT run and reasonably assumes it enforces policy; only by reading `ort/action.yml`’s default *and* `license_compliance.yml`’s opposite setting together can one see that the crown-jewel release path emits signed, SBoM-attested artifacts while the license/policy check is non-blocking and lives in a wholly separate, non-gating workflow.
- **Suggested fix** — Either add `fail-on-violation: true` to the `ort` step in `release.yml`’s `sbom` job (so a violation aborts before `upload-release`/`upload-builds-hex-pm`), or make the release jobs depend on a required status from `license_compliance.yml` (e.g., a shared reusable-workflow `needs:` or branch-protection required check) so a failed license gate blocks publication.
