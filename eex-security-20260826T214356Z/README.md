# Security Scan Summary — Elixir EEx (`elixir/lib/eex`)

**Scan:** `eex-security-20260826T214356Z` · **Target:** EEx, Elixir's embedded-template compiler (5 source files) · **Analysed:** 2026-08-26

**Bottom line:** This is a small, mature, well-understood surface. The cross-file and cluster passes found nothing, and four of five source files are clean. Exactly **one** substantive finding survives — a Medium atom-exhaustion DoS in the tokenizer. The dominant EEx risk (SSTI→RCE from an untrusted template) is by-design and out of scope: EEx's documented contract is *"the template is code, not data."*

---

## Ranked findings

### 1. Atom-table exhaustion via `EEx.tokenize/2` — **Medium** (verified, high confidence)
`compiler.ex:83` · [`CSCAN-compiler-tokenize-atom-exhaustion`](./lib-eex-compiler.ex.md#cscan-compiler-tokenize-atom-exhaustion) · CWE-770

For every `<%…%>` tag, the classifier runs the full Elixir tokenizer over the tag body with default options purely to label it start/middle/end/plain. Classification only inspects fixed keyword token *types* (`:do`, `:end`, `:stab_op`, `:fn`, …), but the default-options tokenizer **interns a permanent, never-GC'd atom for every identifier, alias, and `:atom` literal** as a side effect. This runs inside `EEx.tokenize/2` — the one public, documented entry point that does *not* compile or evaluate the template.

- **Impact:** Any tool that treats `EEx.tokenize/2` as a side-effect-free lexer on untrusted `.eex` input — a formatter, linter, syntax highlighter, LSP, or template-preview — leaks one atom per unique identifier. One large hostile template, or many small ones accumulating (atoms persist across calls), exhausts the ~1M-atom table and aborts the whole BEAM node with `system_limit`. The victim's assumption "lexing doesn't execute anything, so it's safe" is silently false.
- **Why Medium, not High:** the mechanism is certain and empirically reproduced (20,000 unique idents → +20,039 atoms; per-marker controls isolate the sink to line 83), but it depends on the plausible-but-not-universal pattern of tokenizing untrusted templates without running them. On compile/eval paths it adds no marginal risk — those are RCE-by-design already.
- **Fix (with a verified subtlety):** pass a `:static_atoms_encoder` to the line-83 tokenizer call that resolves existing atoms and returns a non-interned placeholder for novel ones. Note: `existing_atoms_only: true` alone is **wrong** — it makes the tokenizer error on any new identifier, misclassifying legitimate blocks like `<% if newvar do %>`.

---

## Latent / ruled-out but worth a note

These were investigated and explicitly *not* filed as findings; recorded here so the coverage is transparent, not as open issues.

- **`EEx.SyntaxError.message/1` echoes raw template bytes (CWE-117 / terminal-escape).** Verified that ESC/BEL bytes from source survive into `Exception.message/1`. Ruled out because the raw snippet is built in `EEx.Compiler` (not `eex.ex`), there is no terminal/log/`~s` sink in scope, and returning the raw diagnostic is idiomatic for every Elixir exception. **Latent defense-in-depth only** — materialises only if a downstream consumer writes it unsanitized.
- **`code_snippet` `String.duplicate` underflow (`compiler.ex:563/570`).** `column: 0` / negative `indentation` yields `ArgumentError` instead of a clean `EEx.SyntaxError`. Ruled out: driven by the *caller-supplied* `:column`/`:indentation` options, not the untrusted-template channel; template content alone always produced a clean error.
- **`insert_quoted` `KeyError` via `__EEX__(N)` in a tag; `String.to_unsafe_atom("arg#{N}")` (`engine.ex:198`).** Compile-path only (already RCE) or bounded by an internal monotonic counter with atoms reused across templates (verified: second same-size template = 0 new atoms). No security uplift.

---

## By-design, out of scope (not defects of this folder)

- **SSTI → RCE.** Feeding attacker-controlled text as the *template* to `eval_string`/`compile_string`/`function_from_*` is arbitrary code execution. This is EEx's documented contract — there is no sandbox. Only `bindings`/`@assigns` is a safe data channel (verified inert). The caller owns template trust.
- **No HTML/output escaping.** `<%= %>` runs `String.Chars.to_string/1` with no escaping; XSS defense belongs to downstream engines (e.g. `Phoenix.HTML`), not EEx.

---

## Coverage notes

- **CROSS_FILE.md:** No cross-file findings. The one genuine coupling — `SmartEngine` delegating its whole security posture to `EEx.Engine` — was verified by reading `engine.ex` and re-executing the atom/marker axes on OTP 29 (deterministic `argN` atoms, non-raising `fetch_assign!`, marker rejection). Atom-exhaustion surfaces are cleanly partitioned (`compiler.ex:83` vs `engine.ex:198`) with no double-count or gap. No "safe" verdict rests on an unverified caller-side check.
- **CLUSTER_lib-eex.md:** No cluster findings.
- **Clean files (no surviving findings), omitted from ranking:** `engine.ex`, `smart_engine.ex`, `eex.ex`, `mix.exs`.
