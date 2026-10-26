# /Users/lukas/code/claude-scanner/elixir/lib/logger/mix.exs
<!-- sha256: 04dee679e4658078338d9191f920edb75a87d24ca0bd4536df7cb8413b45d671 -->

# /Users/lukas/code/claude-scanner/elixir/lib/logger/mix.exs
<!-- sha256: 04dee679e4658078338d9191f920edb75a87d24ca0bd4536df7cb8413b45d671 -->

_Re-analysed on 2026-08-27_

`System.version()` returns a plain binary (`"1.21.0-dev"`) sourced from the Elixir installation itself — no external input flows into it. Every other value in the file is a static literal. Nothing is attacker-reachable.

---

# Security Review — `elixir/lib/logger/mix.exs`

## Summary
This is a trivial 38-line Mix project definition containing only static literals and one pure version accessor; it exposes no attacker-reachable sink. The prior report carried one hedge — a "single-file review" scope note deferring the *behavior* of the sibling Logger source files. On re-review that note is confirmed to be a genuine scope boundary, not an unverified assumption about this file: no disposition in this file depends on the content of a neighbour module. The env keys declared here are inert literals; the modules that *consume* them (`app.ex`, `handler.ex`, `formatter.ex`, `utils.ex`) are what turn a value into behavior, and those are separate review units. No claim about `mix.exs`'s own behavior was left resting on an assumption about another file, so there is nothing to verify against a neighbour or upgrade to a finding.

## Security

No findings.

The complete sink inventory and its disposition:

- **`System.version()` (line 11)** — the sole non-literal expression, evaluated at build time (`compile-time execution` class). It reads the version string of the Elixir installation performing the build; no file, network, dependency, or archive data reaches it. Returns a plain binary. **Confidence: high** (`verified` — executed `elixir -e 'v = System.version(); IO.inspect(v); IO.inspect(is_binary(v))'` → `"1.21.0-dev"` and `is_binary/1` → `true`). Not attacker-influenced.
- **`env: [...]` defaults (lines 20–34)** — sets the `:logger` application-environment defaults (`shared mutable state` class). All 14 values are compile-time literals with safe magnitudes (`truncate: 8096`, `discard_threshold: 500`, `sync_threshold: 20`, `discard_threshold_periodic_check: 30_000`, remaining are `false`/`nil`/`[]`/MFA-literal). None are derived from external input, so no cross-caller poisoning originates here. **Confidence: high.** (The threat-model DoS concerns about `:infinity` truncate / huge thresholds live at the *configuration override* boundary — `configure/1` / `config :logger` — not in these hardcoded defaults.)
- **`mod: {Logger.App, []}`, `registered: [Logger, Logger.Supervisor]`, `translators: [{Logger.Translator, :translate}]` (lines 18–23)** — literal atoms / module-function references (`behaviour dispatch` class). No atoms are created from bytes; no dispatch target is externally influenced. **Confidence: high.**

## Risks / caveats
- The file is self-contained and calls no macro, module attribute, or helper that reads the filesystem or network at compile time, so the build-machine attack surface is nil.
- Security-relevant *behavior* of the Logger application (truncation byte-math, translator crash handling, format-string/ANSI injection, atom creation in `Formatter.compile/1`) lives in the sibling source files (`formatter.ex`, `utils.ex`, `translator.ex`, `app.ex`, `backends/*`). This is a legitimate review-unit boundary, not a hedged assumption about `mix.exs`: this manifest only *declares* default values and module references; it neither reads nor transforms any of them. No finding disposition in this file was deferred to those modules — they consume the declared defaults but cannot change what this static file does. Those modules are reviewed as their own units.
- The one live call was verified behaviorally by execution; the remaining items are static literals whose safety is evident by inspection and require no execution.

### Ruled out
- **Compile-time execution via `System.version/0`** — pure accessor returning the local Elixir version; no external/untrusted input, no file/network access. Not a finding.\
  Axes examined: return type (binary), external-input reachability (none — value originates in the Elixir install, not from build inputs), file/network side effects (none). | Not examined: none.
- **Shared-state write via `env` defaults** — values are static literals, not externally derived; cannot be used by one caller to poison another. Not a finding.\
  Axes examined: literal-ness of each of the 14 values (all compile-time constants), external-derivation (none), magnitude safety of the numeric thresholds. | Not examined: none.
- **Behaviour/atom-dispatch via `mod`/`registered`/`translators`** — all targets are literal atoms present at compile time; no `String.to_atom`/`Module.concat` on external bytes. Not a finding.\
  Axes examined: atom provenance (all literal), dynamic-atom-creation from bytes (none present). | Not examined: none.

## Did I verify this?
- `System.version/0` return type and value: **yes, executed** (`elixir -e '...'` → `"1.21.0-dev"`, `is_binary` → `true`).
- The `env` defaults, `mod`/`registered`/`translators`: **yes, by inspection of the source** — every value on lines 18–34 is a literal with no expression, accessor, or interpolation; no execution is required to establish that a literal is a literal.
- The "single-file" scope note: **re-examined and confirmed as a scope boundary, not an unverified cross-file assumption** — no disposition above rests on the content of a neighbour module, so no neighbour read was needed to clear any axis.

### Dropped in re-review
None. All prior dispositions are preserved; the prior "single-file review" caveat is reframed above as a confirmed review-unit boundary rather than a hedge, with no change to any finding.
