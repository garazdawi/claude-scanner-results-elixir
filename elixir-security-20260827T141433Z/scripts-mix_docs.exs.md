# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/mix_docs.exs
<!-- sha256: 3d52639b12306e32a73a6c1a69997bd308fa81a6fa66c3867b86e1d4016c4575 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/mix_docs.exs
<!-- sha256: 3d52639b12306e32a73a6c1a69997bd308fa81a6fa66c3867b86e1d4016c4575 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

`mix_docs.exs` is the ExDoc build-config script for the non-Elixir apps (EEx, ExUnit, IEx, Logger, Mix); it is a near-identical sibling of `elixir_docs.exs`. In this re-review pass I resolved the single flagged hedge ("would warrant") in the prior report's Mermaid caveat by reading the sibling `elixir_docs.exs` in full and by executing the `canonical` interpolation/trim path in the local Elixir toolchain. Both the SRI finding and every ruled-out entry are unchanged in substance; the speculative future-change hedge has been replaced with the verified present-state facts.

The standing-context callgraph neighbours (`elixir_tokenizer`, `elixir_interpolation`, `elixir_utils`, …) are the Erlang compiler front-end and have **no** call relationship to this ExDoc config script; none of the Unicode/tokenizer obligations in the checklist apply to this file. The only in-repo neighbour that matters is the sibling `elixir_docs.exs`, which I read.

---

## Summary
`mix_docs.exs` is an ExDoc build-config script whose only external input (`CANONICAL`) is build-operator-controlled and never reaches a dangerous sink; the sole security-relevant observation is a third-party CDN `<script>` emitted into the generated docs without Subresource Integrity. The identical `<script>` is present in the sibling `elixir_docs.exs:219` (verified by reading it).

## Security

#### `CSCAN-mix_docs-cdn-script-no-sri`
- **Finding ID:** `CSCAN-mix_docs-cdn-script-no-sri`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `mix_docs.exs:39` (`before_closing_body_tag`, `:html` clause)
- **Class:** Unclassified (supply chain / missing Subresource Integrity)
- **Issue:** The HTML injected into every generated documentation page loads `mermaid@11.6.0` from `https://cdn.jsdelivr.net` with `<script defer src="...">` and **no `integrity=`/`crossorigin=` (SRI) attribute**. There is no integrity check on the fetched code before the browser executes it.
- **Impact / threat:** The generated docs are published (hexdocs.pm pages for EEx/ExUnit/IEx/Logger/Mix) and read by many developers. If jsDelivr serves altered content for that path (CDN compromise, cache poisoning, or MITM against a reader whose transport can tamper with the asset), arbitrary JavaScript runs in every reader's browser in the docs' origin. The trust boundary crossed is between the doc-publishing pipeline's output and third-party CDN infrastructure. The npm version is pinned (immutable), so tag-repointing is not the vector; the residual risk is CDN/delivery-path compromise, which SRI is specifically designed to neutralize. Severity is Low because it requires compromising a reputable CDN and the payload lands only in client browsers viewing docs, not on the build machine.
- **Evidence:** `asserted`. The structural fact is verified by inspection — the emitted `<script>` tag (`mix_docs.exs:39`) carries no `integrity`/`crossorigin` attribute. The identical snippet in `elixir_docs.exs:219` was confirmed by reading that file in this pass (byte-for-byte identical `<script>` line and following block, `elixir_docs.exs:216-251`). The exploit impact (CDN-served code executing unchecked) is reasoned about, not demonstrated, since it depends on external CDN behavior I cannot drive from the sandbox.
- **Suggested fix:** Add an `integrity="sha384-…"` (and `crossorigin="anonymous"`) attribute to the mermaid `<script>` tag, pinned to the sha384 of the exact `mermaid@11.6.0` bundle; browsers will then refuse to execute a tampered payload. Apply the same change to the identical snippet in `elixir_docs.exs:219`.

### Ruled out

- **`canonical` interpolated into `https://*.hexdocs.pm/#{canonical}` URLs (`mix_docs.exs:28-33`) and into search config via `String.trim/2` (line 19)** — the value comes solely from `System.fetch_env!("CANONICAL")` or `System.version()` (verified by reading lines 5-9). I confirmed by execution that the interpolation is raw string substitution with no escaping: pushing `x"><script>/` through the deps template yields `https://eex.hexdocs.pm/x"><script>/` verbatim, and `String.trim("1.18.0/", "/")` yields `"1.18.0"`. So *if* `canonical` were attacker-controlled it would be an HTML/URL-injection sink — but `CANONICAL` is a build-time environment variable. I grepped the entire checkout for any assignment of `CANONICAL` (Makefiles, `*.mk`, CI YAML) and found **none**, i.e. it is supplied only by the external release/docs build, which a party who can set it already controls (they have build-machine code execution). It is not reachable as untrusted input through any documented public API. Basis: **verified** — source of `canonical` traced within the file; injection behaviour and trim behaviour confirmed by running `elixir -e`; absence of an in-repo `CANONICAL` assignment confirmed by grep.
- **`graphEl.innerHTML = svg` after `mermaid.render()` (`mix_docs.exs:59`)** — the rendered diagram source is `codeEl.textContent` from `pre code.mermaid` blocks, i.e. Mermaid diagrams authored in the maintainer-written doc markdown, not untrusted input arriving through this file's API. The ruling rests on that trust boundary (no untrusted-data path reaches this sink), which does not depend on any external library default. As a secondary observation, the `mermaid.initialize({...})` call at lines 45-48 sets only `startOnLoad` and `theme` and does **not** override `securityLevel`, so the Mermaid 11 default applies; I read the sibling `elixir_docs.exs:225-228` and confirmed the initialize call there is identical, so neither config lowers the security level. Basis: **verified** by reading the emitted script (lines 45-63) and the sibling (lines 216-247); trust of diagram source follows from it originating in the repo's own docs.
- **`System.fetch_env!("CANONICAL")` raising when unset (`mix_docs.exs:6`)** — a build-time crash under the operator's own control; not a security issue (no attacker boundary). Basis: verified by reading lines 5-9.
- **The `_ -> ""` fallback clause (lines 69-71)** — reachable and correct (invoked for the `markdown`/`epub` formatters listed at line 35); not dead defensive code. Basis: verified by reading lines 35 and 69-71.
- **No other sink classes present** — the file contains no `Code.eval*`/compile, no external-to-atom conversion (only hardcoded atoms), no `System.cmd`/`os.cmd`/shell, no file/path/archive/deserialization/regex-from-input/crypto/ETS/persistent_term/process-dictionary usage. Basis: verified by full read of the 73-line file.

## Risks / caveats
- **Trust-boundary basis for `CANONICAL`:** this is a partial checkout (`lib/elixir/scripts/…`); the release/docs build that sets `CANONICAL` is not present. I grepped the checkout and found no `CANONICAL` assignment, so its origin is confirmed only to be *outside this checkout* — consistent with the standard Elixir docs-build convention of passing it from the release/docs Makefile. The trust-boundary claim (build-operator-controlled) therefore rests on that convention plus the executed proof that only a build-machine-controlled value reaches the interpolation. This is the load-bearing assumption behind the two `canonical` ruled-out entries; it does not become a finding because no in-repo path feeds `CANONICAL` from untrusted data.
- **Mermaid rendering:** the current config does not lower Mermaid's `securityLevel` (verified in both this file and `elixir_docs.exs`), and the diagram source is maintainer-authored, so the `innerHTML = svg` sink is not reachable by untrusted data in the present code. There is no speculative dependency in this ruling.
- The SRI finding concerns client-side behavior of the *published* docs rather than the build machine; I could not empirically demonstrate a CDN-tamper scenario from the sandbox, so its impact is `asserted`.

## Did I verify this?
- **Read the file** (`mix_docs.exs`, all 73 lines) — yes.
- **Read the callgraph neighbour** — the standing-context neighbours are the Erlang compiler front-end and are unrelated to this ExDoc script; the one relevant in-repo neighbour, `elixir_docs.exs`, I read in full (253 lines). Confirmed the CDN `<script>` is byte-identical at `elixir_docs.exs:219` and the `mermaid.initialize` call is identical (no `securityLevel` override).
- **Executed the `canonical` path** — ran `elixir -e` to confirm raw (unescaped) interpolation into the deps URLs and `String.trim/2` behaviour; this is what demoted the `canonical` axis from "assumed safe encoding" to "verified raw substitution, safe only because the source is trusted."
- **Grepped for `CANONICAL` assignment** across the checkout — none present; documented as the remaining external-trust assumption.

## Dropped in re-review
None. Every prior finding and ruled-out entry is retained. The only change is the prior report's Risks/caveats hedge — *"a future change that both sets `securityLevel: "loose"` and renders untrusted diagram text would warrant re-review"* — which was a speculative statement about hypothetical future code, not an unverified assumption about the present file. It is replaced above with the verified present-state facts (securityLevel not overridden in either sibling; diagram source maintainer-authored). No axis was left untried for the entries that were downgraded to `verified`; the `CANONICAL` origin remains an explicitly-stated external-trust assumption rather than a cleared axis, because the build file that sets it is outside this checkout.
