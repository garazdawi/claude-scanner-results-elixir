# /Users/lukas/code/claude-scanner/elixir/.github/workflows/release_pre_built/action.yml
<!-- sha256: c5cc0381f0cb780bf88cdbae28cfc14eb4c686e4a22a98bbd2c3fd607d4fc80d -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/.github/workflows/release_pre_built/action.yml
<!-- sha256: c5cc0381f0cb780bf88cdbae28cfc14eb4c686e4a22a98bbd2c3fd607d4fc80d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-27T21:10:00Z_

Reachability picture re-confirmed by reading the neighbours directly this pass (not relying on the folder threat model):

- Single caller `release.yml:90` (`build` job). Triggers are `push` to `main`/`v*.*`, tags `v*`, `workflow_dispatch` (`release.yml:6-15`) — **no `pull_request`/`pull_request_target`**. Unprivileged fork-PR authors cannot reach this action.
- Inputs come from a static matrix (`release.yml:73-83`: `otp: 27/28/29`, `otp_version: "27.0"/…`, `build_docs`), not event payload.
- Calling `build` job inherits top-level `permissions: contents: read` (`release.yml:21-22`), no job override, passes no secrets (`release.yml:91-94`).
- **New this pass:** I read `release.yml` fully and confirmed the artifact chain `build → sign(needs:build) → sbom(needs:build,sign) → upload-release/upload-builds-hex-pm`, and I read `ci.yml`/`release_notifications.yml` to confirm the setup-beam version delta. Both formerly-hedged claims are now verified. I also **executed** the `jq`→`GITHUB_ENV` newline mechanism.

## Summary

For its actual (non-fork, push/tag/dispatch-only, static-input) reachability this composite action is well-hardened — inputs flow through `env:` and both actions are SHA-pinned — leaving a third-party-compromise `GITHUB_ENV` injection path (mechanism now executed and confirmed) and a confirmed stale/inconsistent `setup-beam` pin.

## Security

#### `CSCAN-action-hexpm-github-env-injection`
- **Finding ID** — `CSCAN-action-hexpm-github-env-injection`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `release_pre_built/action.yml:58-60`, step "Get ExDoc ref"
- **Class** — Unclassified (`GITHUB_ENV` injection from third-party network response)
- **Issue** — The step fetches `latest_stable_version` from hex.pm, interpolates it into `ref`, and appends it to `$GITHUB_ENV` with no validation:
  ```bash
  ref=v$(curl -s https://hex.pm/api/packages/ex_doc | jq --raw-output '.latest_stable_version')
  echo "EX_DOC_REF=$ref" >> $GITHUB_ENV
  ```
  A JSON response whose `latest_stable_version` value embeds a newline (e.g. `1.0\nEX_DOC_REF=attacker-ref`) writes **two** lines to `$GITHUB_ENV`, letting the responder set arbitrary environment variables for every subsequent step in the `build` job. `EX_DOC_REF` is consumed as the `ref:` of the `elixir-lang/ex_doc` checkout (line 65), so the injected value also steers what code is built.
- **Impact / threat** — Reached only by trusted triggers (push to `main`/`v*.*`, tag `v*`, `workflow_dispatch`); **not** fork-PR reachable. Attacker is a compromised/MITM'd hex.pm ("compromised third-party dependency" model); HTTPS + cert validation makes MITM hard, so this hinges on hex.pm itself being subverted. If it is, injected env vars poison the build environment. **Verified this pass** that those artifacts flow downstream: `release.yml` `build` uploads `build-{linux,windows}-elixir-otp-*`/`Docs` (lines 102-119); `sign` (`needs: [build]`, line 123) downloads and signs them; `sbom` (`needs: [build, sign]`, line 192) and `upload-release` (`needs: [create_draft_release, build, sign, sbom]`, line 278) publish them — so a poisoned build taints the signed, SBOM'd, published release. Large blast radius. Line annotated `# zizmor: ignore[github-env]`, so residual is a known/accepted one.
- **Evidence** — `verified` (reachability): grep shows single caller `release.yml:90`; `release.yml:6-15` has no `pull_request` trigger; `build` job passes no secrets and runs at `contents: read`. `verified` (downstream blast radius): read `release.yml:102-278`, artifact-name dependency chain confirmed. `verified` (injection mechanism, **executed this pass**): I ran the exact pipeline against a crafted response `{"latest_stable_version":"1.0\nEX_DOC_REF=attacker-ref"}` — `jq --raw-output` emitted the embedded newline literally, command substitution preserved it, and `echo "EX_DOC_REF=$ref" >> file` produced two lines (`EX_DOC_REF=v1.0` then `EX_DOC_REF=attacker-ref`, `wc -l` = 2). Since `GITHUB_ENV` is parsed per-line with last-value-wins, the attacker-controlled second line prevails. The only residual `asserted` element is that a live hex.pm compromise is required to deliver such a payload — the reason severity stays Low, not the mechanism.
- **Suggested fix** — Validate against a strict pattern (which also rejects embedded newlines) before writing, and quote `$GITHUB_ENV`:
  ```bash
  ref=v$(curl -sf https://hex.pm/api/packages/ex_doc | jq --raw-output '.latest_stable_version')
  if ! printf '%s' "$ref" | grep -Eq '^v[0-9]+\.[0-9]+\.[0-9]+$'; then
    echo "Unexpected ex_doc version: $ref" >&2; exit 1
  fi
  echo "EX_DOC_REF=$ref" >> "$GITHUB_ENV"
  ```

#### `CSCAN-action-setup-beam-stale-pin`
- **Finding ID** — `CSCAN-action-setup-beam-stale-pin`
- **Severity** — Low
- **Confidence** — high (on the fact); the security impact is latent
- **Location** — `release_pre_built/action.yml:21`, `uses: erlef/setup-beam@5304e04...`
- **Class** — Unpinned third-party actions (stale-pin variant)
- **Issue** — `erlef/setup-beam` is pinned to `5304e04ea2b355f03681464e683d92e3b2f18451 # v1.18.2`. SHA-pinning is satisfied (no mutable-tag risk), but this is an **older** release than the pin used elsewhere in this same repo. **Verified this pass by grepping/reading the neighbour workflows** (not the threat model): `ci.yml:49`, `ci.yml:130`, and `release_notifications.yml:24` all pin `erlef/setup-beam@54075bcc5e249e4758d363f27d099f55d843f124 # v1.24.1`. This file is the sole outlier at v1.18.2. A third-party action on a release-signing pipeline should be current and consistent so any security fix shipped between v1.18.2 and v1.24.1 is picked up.
- **Impact / threat** — No active exploit; the concern is latent old-version exposure plus version drift on a supply-chain-critical (release-building) job. Reachable only via trusted release triggers, not by an outsider.
- **Evidence** — `verified`: line 21 pins v1.18.2 (`5304e04…`); `ci.yml:49,130` and `release_notifications.yml:24` pin v1.24.1 (`54075bcc…`) — confirmed by grep across the folder this pass. No specific CVE for v1.18.2 located, so the security consequence remains asserted/latent.
- **Suggested fix** — Bump to the repo-consistent pin: `uses: erlef/setup-beam@54075bcc5e249e4758d363f27d099f55d843f124 # v1.24.1`.

### Ruled out

- **Expression injection of `inputs.otp` / `inputs.otp_version` into `run:`** — `verified` | Axes examined: value-provenance (static matrix in `release.yml:73-83`), splice-site (every `${{ }}` at lines 23, 33, 49, 50, 52, 62, 65, 69, 77 appears only inside `env:`, `if:`, or a `with:` action input, never in a shell line; shell reads them as quoted vars `"${INPUT_OTP}"`, `"$INPUT_OTP_VERSION"`) | Not examined: none.
- **Fork / `pull_request_target` / `workflow_run` reachability** — `verified` | Axes examined: caller set (grep `release_pre_built` → only `release.yml:90`; `ci.yml` does not use it), trigger set (`release.yml:6-15` = push branch/tag + `workflow_dispatch` only) | Not examined: none. No unprivileged-outsider path; anyone who can reach it already controls the tree via push access.
- **Token permissions / secrets exposure** — `verified` | Axes examined: composite-action permission model (composite actions cannot declare `permissions:`), inherited grant (`release.yml:21-22` top-level `contents: read`, no `build` job override at 66-94), secret passing (caller passes only `otp_version`/`otp`/`build_docs`, no `secrets.*` referenced in this action) | Not examined: none. No secret reaches this code; nothing to escalate or leak.
- **Mutable-tag unpinned actions** — `verified` | Axes examined: both `uses:` are full-SHA pinned (`setup-beam@5304e04…`, `checkout@11bd719…`); ex_doc checkout sets `persist-credentials: false` (line 67) | Not examined: none. (Staleness captured as its own finding.)
- **`echo "$PWD/bin" >> $GITHUB_PATH` (line 31) and `build.sh`/`mix` executing tree code** — `verified` | Axes examined: source of `$PWD` (the `elixir` checkout at a pushed/trusted ref per `release.yml:85-87`, not a fork tree), trigger set (no fork trigger) | Not examined: none. Adding it to `PATH` and running build scripts imports no outsider code. (Annotated `# zizmor: ignore[github-env]`.)
- **HTTP/MITM on the version fetch** — `verified` | Axes examined: URL scheme (line 58 `https://hex.pm/...`, satisfies HTTPS obligation) | Not examined: none. Residual hex.pm-compromise risk is captured in the injection finding above.

## Did I verify this

- **Injection mechanism** — **Executed** this pass in the sandbox against a crafted malicious response; the `jq --raw-output` newline → command-substitution → `GITHUB_ENV` two-line-write behaviour is confirmed by observed output (`wc -l` = 2, attacker line present and last-wins). Only the "live hex.pm must be compromised to deliver the payload" precondition remains a reasoned assertion — that is why severity stays Low.
- **setup-beam staleness** — **Verified by reading neighbours** (`ci.yml:49,130`, `release_notifications.yml:24`) this pass, replacing the first-pass reliance on the folder threat model. The version delta (v1.18.2 vs v1.24.1) is a direct grep fact. Whether v1.18.2 carries a specific published CVE is not confirmed; the finding rests on staleness/inconsistency, not a known CVE.
- **Downstream artifact trust** — **Verified by reading `release.yml`** this pass (the first pass called it "out of scope"): the `build → sign → sbom → upload` name-based artifact dependency chain (`needs:` at lines 123, 192, 278) is confirmed, substantiating the injection finding's blast-radius claim. The internal correctness of the `sign`/`sbom` jobs themselves (e.g. the `${{ github.sha }}` interpolation at `release.yml:401,405`) is genuinely a different file's concern and not adjudicated here.
- **Reachability** — Verified by direct reads of `release.yml` triggers and caller graph; a future workflow invoking this action from a `pull_request`/`pull_request_target` trigger, or feeding event-derived values as inputs, would raise the severity of the input/env handling — the folder-level invariant to re-check on drift.

### Dropped in re-review

None. Both prior findings are preserved (and strengthened), and no ruled-out entry was upgraded away. Two first-pass hedges were resolved: the setup-beam version comparison was upgraded from "per folder threat model" to verified-by-reading, and the downstream-artifact-flow "out of scope" caveat was upgraded to verified-by-reading `release.yml` — both fold into the existing findings rather than creating new ones. The injection finding's mechanism moved from `asserted` to executed evidence, but its Confidence stays **medium** because the exploit still requires a hex.pm compromise.
