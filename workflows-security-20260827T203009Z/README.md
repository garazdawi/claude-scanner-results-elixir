# Security Scan Summary — `elixir/.github/workflows`

**Target:** Elixir language project CI/CD (7 GitHub Actions workflows + 2 local composite actions), commit `3389f84`. This is a supply-chain review, not a wire-protocol one.

**Headline:** No fork-PR–reachable (primary-attacker) vulnerability was found — every secret-bearing and signing path is correctly gated behind `push`/tag/`release`/`workflow_dispatch` triggers with `environment: release`, and the lint/scan workflows that do run fork code hold read-only tokens with no secrets. **The real exposure is the supply-chain trust boundary of the signed release**: it depends on several *unpinned mutable upstreams*, so the impactful attacker here is a **compromised third-party dependency (tertiary)** or a **tag-pushing maintainer (secondary)**, not an outside fork author. All findings below are Medium or Low.

---

## 1. Signed release chain trusts multiple mutable upstreams *(cross-file, Medium, high confidence)*
`CSCAN-crossfile-mutable-upstream-in-signed-release-chain` — files: `release.yml`, `release_pre_built/action.yml`, `ort/action.yml`, `ci.yml`

The **signed, SBOM'd, attested** release produced by `release.yml` (jobs `sign`/`sbom` hold `id-token: write` + `attestations: write` + `contents: write`) is only as trustworthy as the *weakest* of at least three unpinned upstreams that feed artifacts into that trusted boundary:
- `release_pre_built/action.yml:58` fetches `ex_doc`'s version from **hex.pm** and checks that ref out (build → sign → sbom → upload);
- `ort/action.yml:47` checks out `oss-review-toolkit/ort-config@main` (**mutable branch**); and
- `ort/action.yml:94` runs the ORT container by **mutable tag** `ort:92.2.0`.

No per-file report tallied this aggregate: a *single* upstream compromise (hex.pm, `ort-config`, or the ORT registry) lands inside the signing boundary, and the attestation/SBOM then vouches for the poisoned bytes. **Fix:** pin all three to reviewed digests/SHAs. This finding is the framing for items 2–4 below (its constituents).

---

## 2. ORT default config checked out from a mutable `@main` branch *(Medium, high)*
`CSCAN-action-ort-config-mutable-ref` — `ort/action.yml:44-49`

`actions/checkout` is SHA-pinned, but the *content* `oss-review-toolkit/ort-config` is taken at branch `main` and used as the base config (curations, resolutions, package configs) driving the analyzer/scanner/advisor. In `release.yml`'s `sbom` job this config governs the SBOM that is **attested, signed, and published to everyone who installs Elixir** — so whoever controls/compromises that branch could suppress a real advisory or inject a false curation into the signed SBOM. Traced to `release.yml:222 → attest-sbom` (verified by reading). **Fix:** pin `ref:` to a reviewed commit SHA.

## 3. ORT runtime container pinned by mutable tag, not digest *(Medium, high)*
`CSCAN-action-ort-image-mutable-tag` — `ort/action.yml:94`

The ORT CI action is SHA-pinned, but the container it runs is `ghcr.io/oss-review-toolkit/ort:92.2.0` (a re-pushable tag, not `@sha256:`). If that tag is repushed, arbitrary code runs inside the `sbom` job whose downstream steps carry `id-token`/`attestations`/`contents: write` — enough to forge/alter the signed SBOM and OIDC attestation shipped to all installers. **Fix:** pin by `@sha256:` digest.

---

## 4. Unvalidated hex.pm response written to `$GITHUB_ENV` in the release build *(Low, medium)*
`CSCAN-action-hexpm-github-env-injection` — `release_pre_built/action.yml:58-60`

`ref=v$(curl … hex.pm/api/packages/ex_doc | jq … .latest_stable_version)` is appended to `$GITHUB_ENV` with no validation; a value containing a newline writes a **second** env line, letting the responder set arbitrary env vars for every later step in the `build` job — and `EX_DOC_REF` steers the `ref:` of the `ex_doc` checkout that is then compiled. **The injection mechanism was executed and confirmed** (`wc -l` = 2, attacker line wins). Bounded to Low only because it requires a hex.pm compromise/MITM; the poisoned build then flows into the signed/published release (large blast radius). Line is annotated `# zizmor: ignore[github-env]` (accepted). **Fix:** validate against a strict `^v\d+\.\d+\.\d+$` pattern and quote `$GITHUB_ENV`.

---

## 5. ⚠ Coverage gap — `notify.exs` is unreviewed, and forwards the release tag + fetched body into credentialed sinks *(cluster, Low, high; flagged as the #1 follow-up)*
`CSCAN-cluster-notify-tag-taint-unvalidated` — `release_notifications.yml:30-34` → `.github/workflows/notify.exs:6,14,36,45,48,64-66`

`release_notifications.yml` safely passes `$GITHUB_REF_NAME` (double-quoted) and defers the rest to `notify.exs` — **but no per-file report for `notify.exs` exists** (an Unresolved ledger row in `CROSS_FILE.md`, and both `CROSS_FILE.md` and `CLUSTER_root.md` call it the *single highest-value follow-up*). The cluster reviewer read the script directly: `[tag] = System.argv()` is interpolated **without validation** into a `Req.get!` URL path, the Postmark email `Subject`/`HtmlBody` (sent with `ELIXIR_LANG_ANN_TOKEN`), and the Elixir Forum post `title`/`raw` (sent with `ELIXIR_FORUM_TOKEN` under the official "Elixir" username); the forum `raw` also embeds `release["body"]` fetched fresh from `api.github.com`. 

Rated **Low** because grep confirmed **no shell/`eval`/`System.cmd` sink** (this corrects the threat model's "into URLs *and shell*" claim — the sinks are fixed-host HTTPS URLs and JSON bodies), all hosts are HTTPS, and tokens come from env; the reachable attacker is a privileged **tag-pusher/maintainer**, not a fork outsider. **Fix:** validate `tag` against a version regex at the top of `notify.exs` and stop forwarding raw `release["body"]`/`tag` verbatim. **Recommended action: obtain a dedicated `notify.exs` review** — its sanitization is currently verified by no one.

---

## 6. Latent `sed`-script command injection on ORT's `version` input *(Low, high; verified RCE mechanism)*
`CSCAN-action-version-sed-injection` — `ort/action.yml:71-79`

`inputs.version` is passed via `env:` (avoiding *direct* shell injection) but `${ELIXIR_VERSION}` is interpolated verbatim into four `sed s///` scripts. **Verified by execution**: a crafted value using the GNU-sed `e` flag created `PWNED_C` (RCE), and a stray delimiter aborts the step (DoS). Not outsider-reachable through today's two callers (`license_compliance.yml` passes 40-hex `github.sha`; `release.yml` passes push-gated `ref_name`), so it is a defense-in-depth defect in a reusable, release-critical action. **Fix:** reject `version` values containing `/ | ' \ &`/newlines before use.

## 7. `ci.yml` clones and executes `ex_doc@main` on every push/PR *(Low, high; widest reachability, contained)*
`CSCAN-ci-exdoc-unpinned-main-exec` — `ci.yml:80-87`

The docs matrix cell `git clone`s `elixir-lang/ex_doc` at the mutable tip of `main` and compiles/runs it — on `push`, `workflow_dispatch`, **and fork `pull_request`**. This is the one path an outside fork PR can reach, but containment is strong (`contents: read`, no secrets, ephemeral runner), so an ex_doc compromise yields only read-only-token compute; the marginal concern is that it would also execute in trusted maintainer `push` builds. **Fix:** pin the clone to a reviewed commit SHA.

---

## 8. Lower-severity hardening items

| Finding | Sev/Conf | Location | Essence |
|---|---|---|---|
| `CSCAN-action-setup-beam-stale-pin` | Low / high | `release_pre_built/action.yml:21` | `erlef/setup-beam` pinned to **v1.18.2** on the release pipeline while `ci.yml` (49,130) and `release_notifications.yml` (24) use **v1.24.1** — sole stale outlier; bump for consistency and to pick up fixes. |
| `CSCAN-release-github-sha-run-interp` | Low / high | `release.yml:401,405` | `${{ github.sha }}` spliced directly into a `bash run:` (the one break from the file's quoted-`$ENV` pattern). Not exploitable (40-hex), pure hygiene next to secret-bearing AWS/Fastly steps; route via `env:`. |
| `CSCAN-release-sign-sbom-contents-write` | Low / med | `release.yml:136-139,195-199` | `sign` and `sbom` jobs grant `contents: write` but write no contents; excess grant widens blast radius if a pinned action were compromised. Drop to least privilege. |
| `CSCAN-action-scancode-cache-key` | Low / low | `ort/action.yml:84-88` | ScanCode cache keyed on `runner.os` alone (no content hash). Poisoning into the signed `sbom` path is currently blocked only by GitHub cache branch-scoping — a platform invariant the reviewer could not execute. Add a content/version dimension to the key. |
| `CSCAN` zizmor-action unaudited-at-SHA | Low / low | `codeql.yml:55` | `zizmorcore/zizmor-action` is correctly SHA-pinned, but the content at that SHA (and whether it matches `v0.6.2`) can't be verified offline; runs with `security-events: write` on PR. Recorded for a networked pass to diff the SHA against the tag. |

---

## Clean files (no surviving findings)
`license_compliance.yml`, `markdown.yml`, `posix_compliance.yml`, and `release_notifications.yml` were each reviewed and ruled clean. `license_compliance.yml` and `release_notifications.yml` correctly **hand their residual risk off** to `ort/action.yml` and `notify.exs` respectively (reflected in items 2–3 and 5) rather than clearing it by assumption.

## Open assumptions worth noting
- Several dismissals rest on **asserted GitHub platform invariants** that could not be executed in the sandbox: fork `pull_request` → read-only token/no secrets, `github.sha` always 40-hex, runner ephemerality, and cache **branch-scoping** isolation (item 8). These are documented behaviors, not defects, but they are load-bearing for the "outsider-contained" conclusion.
- The biggest single gap is **item 5** — a dedicated `notify.exs` review is the highest-value next step.
