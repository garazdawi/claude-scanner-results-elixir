# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/elixir_docs.exs
<!-- sha256: e536dba82126a6d5e047e9965ff883a20c8b8d9f9467372b01c95fb688be1b7d -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/elixir_docs.exs
<!-- sha256: e536dba82126a6d5e047e9965ff883a20c8b8d9f9467372b01c95fb688be1b7d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

This file is an ExDoc build-time configuration script, not part of the compiler front-end the standing threat model describes; none of the tokenizer/interpolation/Unicode obligations apply to it. Its only relevant "callgraph neighbour" present in the checkout is the sibling `mix_docs.exs`, which I read. Its only external input is the `CANONICAL` env var; its only outward-facing product is an HTML fragment emitted into every generated docs page. I re-ran the `CANONICAL` flow across additional axes and read `mix_docs.exs` to settle the hedges from the first pass.

## Summary

`elixir_docs.exs` is a build-time ExDoc config script whose only external input (`CANONICAL`) is build-environment-controlled (operator-set), so it crosses no defended trust boundary. One Low-severity supply-chain hardening gap remains: it embeds a remote CDN `<script>` into generated documentation without Subresource Integrity.

## Security

#### `CSCAN-elixir_docs-cdn-script-no-sri`
- **Finding ID:** `CSCAN-elixir_docs-cdn-script-no-sri`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/elixir/scripts/elixir_docs.exs:219` (`before_closing_body_tag` `:html` clause)
- **Class:** Network (third-party script inclusion / supply chain, CWE-829)
- **Issue:** The `:html` branch emits `<script defer src="https://cdn.jsdelivr.net/npm/mermaid@11.6.0/dist/mermaid.min.js"></script>` with no `integrity=` (Subresource Integrity) hash and no `crossorigin` attribute. ExDoc embeds this fragment verbatim into every generated HTML page, so the browser executes whatever bytes the CDN returns for that URL with no content verification.
- **Impact / threat:** The generated pages are the official Elixir documentation, read by a very large developer audience. If jsdelivr (or an edge cache / DNS path in front of it) is compromised or poisoned for that URL, arbitrary JavaScript runs in the docs' origin for every reader — third-party-script XSS via a trust dependency this file introduces (boundary: generated-docs origin ↔ external CDN). An `integrity` hash would make the browser refuse a tampered script. The version is pinned (`mermaid@11.6.0`) and npm tarballs for an exact version are immutable, so realistic exploitation requires a CDN-infrastructure compromise rather than a routine mutable-`latest` swap — hence Low.
- **Evidence:** `asserted`. The missing-SRI attribute is directly visible and verified in source (`elixir_docs.exs:219`). The conditional CDN-compromise exploitation path is inherently not runnable in this sandbox (it requires a browser and a compromised CDN); this is a defense-in-depth characterization, not a claim I could execute. The finding itself — the absent attribute — is verified by reading.
- **Cross-file note (verified, not assumed):** I read `mix_docs.exs` and confirmed byte-identical tag at `mix_docs.exs:39`. That is a separate file with its own review scope; the same fix applies there. This is a statement of fact from reading the sibling, not a deferred assumption.
- **Suggested fix:** Add a pinned SRI hash and CORS mode, e.g. `<script defer src="…mermaid@11.6.0/dist/mermaid.min.js" integrity="sha384-…" crossorigin="anonymous"></script>`, or vendor `mermaid.min.js` into the docs assets and serve it same-origin.

### Ruled out

- **`CANONICAL` env var → `deps:` URL interpolation (`:86-90`), `String.trim` (`:20`), empty-value branch (`:7-8`).** *Not a finding.* `CANONICAL` is a build-time environment variable set by the Makefile/CI operator; per the folder threat model an operator who sets env vars already has code execution, so this crosses no defended boundary.
  - **Axes examined:** (1) *code-injection* — ran `CANONICAL='#{:os.cmd(~c"id")}'`; output `https://eex.hexdocs.pm/#{:os.cmd(~c"id")}` and `trim=#{:os.cmd(~c"id")}` — the value is a literal runtime string, never re-parsed as Elixir. (2) *host-redirection* — ran `CANONICAL='evil.com/x/'`; output `https://eex.hexdocs.pm/evil.com/x/` — scheme+host are literal source text, the value lands only in the URL path, no host takeover. (3) *HTML/quote-metacharacter passthrough* — ran `CANONICAL='"><script>alert(1)</script>/'`; output `https://eex.hexdocs.pm/"><script>alert(1)</script>/` — **this file does not escape metacharacters**; the raw string is passed into the deps URL. (4) *empty-value* — ran `CANONICAL=''`; output `1.21.0-dev/`, the `System.version()` fallback fires as intended.
  - **Not examined:** whether ExDoc escapes the deps URL when rendering it into a generated page (would decide whether axis 3 could become HTML injection under an *untrusted* `CANONICAL`). ExDoc is not present in this partial checkout, so this axis cannot be tried here. It does not change the verdict: even if unescaped, `CANONICAL` is operator-controlled, so the boundary is undefended by design. If a downstream caller ever fed `CANONICAL` from an untrusted (e.g. PR-controlled) source, axis 3 shows the raw metacharacters would reach ExDoc unescaped — that scenario is a caller/consumption change, not a defect in this file.

- **`System.fetch_env!("CANONICAL")` raising when unset (`:7`).** *Not a finding.* Availability/robustness gate on operator-controlled input, not a security defect.
  - **Axes examined:** *unset-variable* — `System.fetch_env!/1` raises `System.EnvError` when the key is absent, aborting the build loudly (documented `fetch_env!` semantics; the `""` case is the empty-string-set branch, distinct from unset).
  - **Not examined:** none.

- **Client-side `graphEl.innerHTML = svg` in the mermaid renderer (`:239`).** *Not a finding for this file.* Ruled out on data-flow grounds: the inserted `svg` comes from `mermaid.render` applied to diagram text authored in the Elixir docs markdown (trusted contributor content), which is **not an input to this configuration script**. No external input to *this* file reaches this sink.
  - **Axes examined:** *data-flow / reachability* — traced the sink's source: `graphDefinition = codeEl.textContent` from `pre code.mermaid` blocks in rendered docs, i.e. repo markdown, not `CANONICAL` or any env/argv input to this script.
  - **Not examined:** the runtime XSS-sanitization behavior of mermaid's default `securityLevel: "strict"` against a hostile diagram definition — this requires a browser + mermaid runtime, not available in this sandbox. It does not affect the verdict because the sink is unreachable from any input this file controls; sanitization strength would only matter to whoever authors docs markdown (a different review scope).

- **Static `extras`/`assets`/`groups_for_*` paths and `~r"…"` regexes (`:28-111`).** *Not a finding.* 
  - **Axes examined:** *literal-vs-tainted* — all are hardcoded literals compiled from source; no env/argv/file data flows into any path or regex. No traversal or ReDoS surface.
  - **Not examined:** none.

## Risks / caveats

- **SRI exploitability is conditional.** The CDN finding depends on a jsdelivr/CDN-path compromise; under normal operation the pinned immutable version is served intact. Reported as a defense-in-depth hardening gap, not an active vulnerability.
- **Partial checkout.** Only `lib/` is present; the Makefile/CI that sets `CANONICAL` and `Code.eval_file`s this script, and the ExDoc renderer that consumes its output, are absent. The build-time-config trust model is corroborated by the byte-identical sibling `mix_docs.exs` (read) and standard ExDoc tooling. This does not weaken any verdict above, since every ruled-out entry rests on `CANONICAL` being operator-controlled, which the threat model establishes independently of the missing files.

## Did I verify this?

- **SRI finding (line 219):** Verified in source by reading — the tag lacks `integrity`/`crossorigin`. The exploitation path is inherently not executable here (browser + compromised CDN); characterized as `asserted`, not claimed as run.
- **Sibling identical tag:** Verified by reading `mix_docs.exs:39` (byte-identical). The first-pass "out of scope for this file" hedge is now a verified cross-file fact, not an assumption.
- **`CANONICAL` code-injection / host-redirection / HTML-metacharacter / empty-value axes:** Verified by execution — commands and outputs quoted above. The first-pass "would warrant re-rating" hedge is resolved: I ran the metacharacter axis and folded the result into the ruled-out entry's `Axes examined`/`Not examined` fields. One axis (ExDoc's own escaping) remains untried because ExDoc is absent from the checkout; it is explicitly listed as `Not examined` and does not change the operator-controlled verdict.
- **`innerHTML = svg` sink:** Ruled out by verified data-flow tracing (source is repo markdown, not an input to this file). Mermaid's runtime sanitization is listed as `Not examined` (needs a browser) but is not load-bearing for the verdict.

### Dropped in re-review

None. Every prior ruled-out entry and the single finding are preserved; the hedges were resolved in place by executing the `CANONICAL` axes and reading `mix_docs.exs`, not by dropping any conclusion. No prior dismissal was upgraded to a finding, because the added executed evidence (axes 1–4) and the operator-controlled trust boundary continue to support each dismissal.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`CANONICAL` env var → `deps:` URL interpolation (`:86-90`) and `String.trim` (`:20`).** *Not a finding.* Verified by running the exact logic across hostile values: the value lands only in the URL **path** (the scheme+host `https://eex.hexdocs.pm/` is literal source text), so no host redirection is possible; and `#{…}` in the env value is treated as a literal runtime string, not re-parsed as Elixir, so no code injection. Command: `CANONICAL='#{:os.cmd(~c"id")}' elixir -e '… IO.puts("https://eex.hexdocs.pm/#{c}")'` → output `https://eex.hexdocs.pm/#{:os.cmd(~c"id")}` (literal, not executed); `CANONICAL='evil.com/'` → `https://eex.hexdocs.pm/evil.com/` (still hexdocs.pm host). Per the folder threat model, the build operator who sets env vars already has code execution, so this crosses no defended boundary.
