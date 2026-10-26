# Bugfix Scan Summary — Elixir `.github/workflows`

_Source: `results-elixir/workflows-bugs-20260828T075203Z` (13 report files). No `VALIDATION.md` present; ranking is by severity × confidence, with cross-file/cluster findings weighted up. This is a `.github/`-only checkout, so several dependencies (`Makefile`, `.markdownlint-cli2.jsonc`, third-party pinned actions, `notify.exs`) were unreadable and produced low-confidence "unresolved" findings rather than dismissals._

## Confirmed / high-impact

### 1. ShellCheck failures silently swallowed in POSIX-compliance gate — **High, verified**
`posix_compliance.yml:44-53`. Each check is written `shellcheck <file> && echo "…compliant"`. Under the runner's default `bash -eo pipefail`, errexit exempts any command that is *not last* in an `&&` list, so a non-zero `shellcheck` on `bin/elixir` or `bin/elixirc` (2 of the 3 targets) is ignored and the step falls through — only the final `shellcheck bin/iex` can fail the job. The workflow's entire reason for existing (catch POSIX regressions in the launcher scripts) is defeated for two of three files, with a green check. Reproduced in-sandbox. **Fix:** drop the `&& echo` gating so each `shellcheck` runs on its own line.

### 2. Release path is not gated on license/policy compliance — **Medium, high (cross-file cluster)**
`CLUSTER_root.md`: `release.yml`'s `sbom` job invokes the `ort` composite (lines 220-225) **without** `fail-on-violation`, so it inherits `ort/action.yml:13`'s `default: false` and ORT never fails on a violation. The standalone `license_compliance.yml:39` *does* set `fail-on-violation: true`, but `release.yml`'s DAG never depends on it, and both fire on the same tag push. Net result: the crown-jewel release path signs, SBOM-attests, and publishes artifacts to the GitHub Release and Hex while the license/policy check is non-blocking and runs in a wholly separate workflow. Only visible by reading the composite's default *and* both callers together. **Fix:** add `fail-on-violation: true` to the release `ort` step, or make release jobs depend on a required license-gate status.

### 3. Unsigned Windows binaries can ship into an attested release — **Medium, high**
`release.yml:146-165`. Azure login and the code-signing step are both gated on `if: … && vars.AZURE_TRUSTED_SIGNING_ACCOUNT_NAME`. If that repo variable is empty/unset/renamed, GitHub's `&&` yields falsy and **both steps silently skip** — yet "Create Release Hashes" still hashes the unsigned `.exe`, the `sbom` job still attests it, and `upload-release` still publishes it, with no failure signal. There is no fork-vs-canonical distinction. **Fix:** on the release path, fail loudly when signing is expected but unavailable (or guard the skip on `github.repository`).

### 4. ScanCode cache key can never invalidate → stale-but-green license scans — **Medium, high (mechanism)**
`ort/action.yml:84-88`. Cache key is `${{ runner.os }}-scancode` — no ORT/ScanCode version, image digest, or content hash. `actions/cache` restores-only on an exact hit, so once populated the ScanCode index is frozen even when the ORT image (`ort:92.2.0`) is bumped. Because this SBOM feeds the signed/attested release path — and this is the one gate `license_compliance.yml` relies on to block — a stale index can miss a real violation while reporting green (corroborated independently in `license_compliance.yml.md`). **Fix:** fold the ORT image tag/digest into the cache key. *(Caveat: impact is contingent on the configured scanner actually being ScanCode-backed — see #11.)*

### 5. `sed` version substitution corrupts SBOM metadata on `&` — **Medium, high corruption / low likelihood**
`ort/action.yml:71-79`. `inputs.version` is interpolated unescaped into `sed` replacement text, where `&` expands to the whole match. Reproduced: `ver=a&b` silently yields a corrupted `versionInfo`. Delimiter collisions (`/`, `|`) fail loudly and safely; only `&` is silent. Caller analysis bounds real-world likelihood to low — the value is always a `github.sha` (`[0-9a-f]{40}`) or a maintainer-pushed `v*` tag — but the silent-corruption class survives. **Fix:** escape `&`/`\`/delimiter, or template the version via `envsubst`/`awk` instead of a regex replacement.

### 6. `build_docs` feature flag cannot be turned off — **Medium, medium**
`release_pre_built/action.yml:52,62,69,77`. Doc steps are gated on `if: ${{ inputs.build_docs }}`. Composite inputs are strings and GitHub's boolean coercion is "non-empty ⇒ true," so passing the documented `build_docs: false` (string `"false"`) is truthy and docs build anyway — only an omitted/empty value is falsy. The sole current caller (`release.yml`) dodges this via a sentinel-string convention, so the bug is latent but lives in the public input surface. **Fix:** gate on `== 'true'`, or document/default the sentinel convention.

## Lower-impact but confirmed

### 7. Systemic "green gate that verifies nothing" class — **Medium, medium (cross-file)**
`CROSS_FILE.md` observes the same degrade-to-green sink recurring across independent gates with no aggregate guard: the verified `posix_compliance` case (#1), plus unverified instances in `markdown.yml` (empty globs → lint nothing), `codeql.yml` (zizmor may exit 0 on findings), and the frozen ScanCode cache (#4). Treat the *class* — assert non-empty work in each gate (`failglob`/match counts, explicit zizmor severity + non-zero exit, content-hashed cache keys).

### 8. Fastly surrogate key keeps its `.zip` suffix — **Low, verified**
`release.yml:351`: `surrogate_key=${dest/.zip$/}` uses glob replacement where `$` is literal, so no strip occurs; correct form is `${dest%.zip}`. In-pipeline purge stays self-consistent (stored key and purge key share the wrong value), so blast radius is limited to any external Fastly config expecting the extension-less key and to inconsistency with the docs/oldest keys. Reproduced.

### 9. Windows CI compiles Erlang without `warnings_as_errors` — **Low, high**
`ci.yml`: `ERLC_OPTS: "warnings_as_errors"` is set only as `test_linux` job env (line 43); the workflow-level `env` (12-15) exports `ELIXIRC_OPTS` but not `ERLC_OPTS`, and `test_windows` has no `env` block. The two OS jobs enforce different strictness; new Erlang warnings pass green on Windows (still caught on Linux, hence Low). **Fix:** promote `ERLC_OPTS` to workflow-level `env`.

### 10. `notify.exs` robustness never adjudicated — **Low, medium (coverage gap)**
`CROSS_FILE.md`: `release_notifications.yml.md` and `THREAT_MODEL.md` both defer concrete `notify.exs` defects (`MatchError` on `"." <> type = Path.extname(name)` for an extensionless asset name; unchecked per-asset `Req.get!` bodies; unvalidated checksum splitting) to "notify.exs's own review" — but **no `notify.exs` report exists** in this set, though it is the only Elixir file and processes partially attacker-influenceable GitHub-API asset names. The deferral resolves to nowhere. **Fix:** produce a dedicated `notify.exs` review.

## Low-confidence / unresolved externals (recorded, not actionable without the missing artifacts)

- **`--scanners Provenant` may be an invalid/mis-typed ORT scanner id** (`ort/action.yml:111`) — if not ScanCode-backed, silent under-scanning and #4's cache is dead weight. Registry lives in the external ORT image.
- **ORT run-list trailing empty element** when `upload-reports != 'true'` (`ort/action.yml:104`) — parsing behavior lives in the external pinned action.
- **zizmor gate may exit 0 on findings** (`codeql.yml:54-55`) — depends on the pinned action's default exit semantics; unfetchable.
- **markdown lint may be a silent no-op** (`markdown.yml:40-41`) — depends on `.markdownlint-cli2.jsonc`, absent from the checkout.
- **`bom.*` release-upload glob may silently drop SBOM files** (`release.yml:314`) — SBOM basenames set inside the unreadable `ort-ci-github-action`.
- **`COVER=""` present-but-empty on non-coverage CI legs** (`ci.yml:75-76`) — at worst runs coverage on all legs (perf, not correctness); depends on the absent `Makefile`'s gate direction.

_The `CROSS_FILE.md` dependency ledger records **no Contradicted** entries; every in-scope neighbour dependency was corroborated, and all remaining hedges point at genuinely unreadable external artifacts (Unresolved)._
