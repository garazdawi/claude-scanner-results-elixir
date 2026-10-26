# Cross-file findings

## Asserted-dependency ledger

The per-file reports do not use the literal `* Depends on:` syntax; every asserted dismissal is expressed as a prose hedge, a re-review "neighbour now read" note, or a `THREAT_MODEL.md` cross-file invariant. I enumerated all of them below. "Confirmed" means the cited neighbour is in-scope and its report (or the reviewer's direct read of it) substantiates the contract; "Unresolved" means the cited artifact is genuinely outside the extracted tree (external pinned action, absent `Makefile`/`.jsonc`, or a file with no report) and nobody in this set actually checked it.

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| ci.yml | `Makefile` `cover` target gate direction (`COVER=""` behaviour) | Unresolved | `Makefile` not in checkout; ci.yml already carries this as its own low-conf `CSCAN-ci-cover-empty-string-env`. |
| ci.yml | GitHub default-shell contract (bash `-eo pipefail` / pwsh) | Confirmed | Verified against documented GitHub-hosted runner platform contract; posix_compliance.yml relies on the same fact. |
| ci.yml | `erlef/setup-beam` resolving `master`/`maint` refs fails loudly | Confirmed | Self-contained; no `continue-on-error` on setup step. |
| codeql.yml | `zizmorcore/zizmor-action` default exit semantics (does gate turn red) | Unresolved | External action behind a SHA; not fetchable. Kept as codeql.yml's own low-conf finding. |
| codeql.yml | `github/codeql-action` default failure/permission semantics | Unresolved | External action; asserted from prior knowledge, not verified. |
| license_compliance.yml | `ort/action.yml:105` `fail-on` gate consumes `fail-on-violation: true` | Confirmed | ort-action report line 291 + license report line 147 both read line 105; gate fires. |
| license_compliance.yml | `ort/action.yml` `sed` is metacharacter-safe for `github.sha` | Confirmed | ort-action `CSCAN-action-sed-version-ampersand-corruption` confirms 40-hex SHA has no metachars. |
| license_compliance.yml | `ort/action.yml:88` ScanCode cache key lacks content hash | Confirmed | ort-action `CSCAN-action-scancode-cache-key-never-invalidates` (Medium) substantiates. |
| license_compliance.yml | transitive `uses:` in `ort/action.yml` are SHA-pinned | Confirmed | ort-action "Ruled out — Unpinned actions" confirms all three SHA-pinned. |
| markdown.yml | `.markdownlint-cli2.jsonc` defines a covering `globs` | Unresolved | File absent from extracted tree; markdown.yml already carries this as its own low-conf finding. |
| ort/action.yml | `version` input provenance = `release.yml:225` tag / `license_compliance.yml:41` `github.sha` | Confirmed | Both callers read; release.yml:225 and license:41 confirmed. |
| ort/action.yml | stale SBOM flows into `release.yml` `sbom` attest path | Confirmed | release.yml report confirms `sbom` job (needs `[build,sign]`) consumes ort outputs and `attest-sbom`. |
| ort/action.yml | `--scanners Provenant` is a valid ORT scanner id | Unresolved | Scanner registry lives in external `ort:92.2.0` image; not readable. ort.yml's own low-conf finding. |
| ort/action.yml | `ort-ci-github-action` tolerates trailing-empty list element | Unresolved | Parsing lives in external pinned action; not readable. ort.yml's own low-conf finding. |
| ort/action.yml | only `license_compliance.yml` (not `release.yml`) sets `fail-on-violation: true` | Confirmed | Matches `THREAT_MODEL.md` invariant; deliberate design, not a defect. |
| release.yml | `release_pre_built/action.yml:30,47` emit `elixir-otp-${otp}.zip`/`.exe` | Confirmed | release.yml reviewer read the composite directly; basenames match all consumer globs. |
| release.yml | `.sigstore` files land at download root for upload globs | Confirmed | Verified within release.yml itself (sbom-job naming at 245-247). |
| release.yml | `bom.*` matches SBOM basenames emitted by `ort-ci-github-action` | Unresolved | Basenames set inside external action; passthrough in `ort/action.yml`. release.yml's own low-conf `CSCAN-release-sbom-bom-glob-may-be-empty`. |
| release_notifications.yml | `notify.exs:6,16-18,55,73` env/argv contract, draft guard, secret set, no token echo | Confirmed | Reviewer read `notify.exs` directly; arity + secrets + guard confirmed. |
| release_notifications.yml | `notify.exs` downstream robustness (MatchError on extensionless asset, unchecked `Req.get!` bodies) is "notify.exs's own review" | Unresolved | No `notify.exs` report exists in this set; the deferred findings were never minted anywhere. |
| release_notifications.yml | `GITHUB_REF_NAME` == release tag name | Confirmed | Documented GitHub `release`-event ref mapping. |
| release_pre_built/action.yml | sole caller `release.yml:72-83,94` uses sentinel-string `build_docs`, not boolean | Confirmed | release.yml matrix read; `build_docs` set only on OTP-28 row, passed through. |
| THREAT_MODEL.md | `sbom`/`upload` jobs assume `release_pre_built` produced `elixir-otp-${otp}.{zip,exe}` | Confirmed | release.yml read the composite; naming matches. |
| THREAT_MODEL.md | ORT `fail-on-violation` default `false` ⇒ release path does not block on license | Confirmed | ort-action line 291 confirms release.yml omits the input. |
| THREAT_MODEL.md | `notify.exs` relies solely on its `raise "cannot notify a draft release"` guard | Confirmed | release_notifications report verified guard at notify.exs:16-18. |
| THREAT_MODEL.md | `posix_compliance.yml`/`markdown.yml` `paths:` filters are not a safety net for `bin/` in most PRs | Confirmed | posix report confirms filter lists exactly `bin/elixir,elixirc,iex` + workflow. |

No row resolved to **Contradicted**: every in-scope neighbour dependency was corroborated by the cited report, and every remaining hedge points at a genuinely unreadable external artifact (Unresolved). Per the ledger rule, the Unresolved rows are recorded, not minted into findings.

## Findings

#### `CSCAN-crossfile-notify-exs-unreviewed-robustness-gap`
- **Finding ID** — `CSCAN-crossfile-notify-exs-unreviewed-robustness-gap`
- **Severity** — Low
- **Confidence** — medium
- **Files involved** — `release_notifications.yml.md`, `THREAT_MODEL.md`
- **Issue** — `release_notifications.yml.md` (lines 474, 483) explicitly defers three concrete `notify.exs` robustness defects — `MatchError` on the `"." <> type = Path.extname(name)` match at `notify.exs:32` for an extensionless asset name, unchecked bodies from the per-asset `Req.get!` at `notify.exs:26`, and checksum string-splitting without validation — to "`notify.exs`'s own review." `THREAT_MODEL.md` (lines 571, 580) independently lists the same class as a live failure mode ("silent corruption … or a `MatchError` on `[checksum | _]`/`"." <> type`"). **But there is no `notify.exs` report in this set** — the only Elixir file in the folder, and the one processing GitHub-API-derived (partially attacker-influenceable) asset names and bodies, was read only as a neighbour and never received its own review. The deferral therefore resolves to nowhere: both halves name the same file:lines, and neither the workflow review nor any other report actually adjudicates them.
- **Resolution** — Produce a dedicated `notify.exs` review that rules on the `notify.exs:26/32` robustness items the workflow reviewer explicitly punted, rather than leaving them as a cross-file deferral with no destination. This is a coverage gap, not a proven defect — hence Low.

#### `CSCAN-crossfile-silent-green-gate-class`
- **Finding ID** — `CSCAN-crossfile-silent-green-gate-class`
- **Severity** — Medium
- **Confidence** — medium
- **Files involved** — `posix_compliance.yml.md`, `markdown.yml.md`, `codeql.yml.md`, `ort-action.yml.md`, `license_compliance.yml.md`
- **Issue** — Folder-level systemic pattern (reported once): the same sink class — *a CI check that reports green while verifying nothing* — recurs across independent gate workflows, and no aggregate guard exists to catch it. The one **verified** instance is `posix_compliance.yml:44-53`, where `shellcheck … && echo` swallows violations in `bin/elixir`/`bin/elixirc` under errexit (High, reproduced). The same shape appears unverified elsewhere: `markdown.yml:40-41` (no `globs` → lint-nothing exit 0), `codeql.yml:54-55` (zizmor gate may exit 0 on findings), and `ort/action.yml:88` (frozen ScanCode cache → stale-but-green license scan) — the last of which `license_compliance.yml.md` (line 141) flags as load-bearing because it is the sole `fail-on-violation: true` blocking gate. Each per-file review saw its own instance; the cross-file observation is that this pipeline's "safety" checks share a common degradation-to-green failure mode and nothing cross-checks that any of them actually ran with teeth.
- **Resolution** — Treat the class, not just the instances: assert non-empty work in each gate (e.g. `shopt -s failglob`/explicit match count for globs, explicit `min-severity` + confirmed non-zero exit for zizmor, content-hashed cache keys), and confirm the verified `posix_compliance.yml` fix lands. No new source read is required for the confirmed instance; the others remain low-confidence until their external actions/configs are read.
