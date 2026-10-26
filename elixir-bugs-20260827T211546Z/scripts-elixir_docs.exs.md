# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/elixir_docs.exs
<!-- sha256: e536dba82126a6d5e047e9965ff883a20c8b8d9f9467372b01c95fb688be1b7d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T00:04:35Z_

# Bug-hunting review — `elixir_docs.exs`

## Summary
This file is a static ExDoc configuration script that `System.fetch_env!/1`-reads one env var and otherwise returns a literal keyword list; it exposes no public API and no untrusted-input path, and its one piece of runtime logic behaves correctly.

## Bugfixes

No findings.

### Ruled out

- **`System.fetch_env!("CANONICAL")` raises when `CANONICAL` is unset (lines 6–10).** This is by design: the file is only ever evaluated through the docs build (Elixir's `Makefile`, which exports `CANONICAL` before invoking `mix docs`), never as a public API. A hard failure on missing required config is the intended contract, not a bug. Basis: `verified` — with `CANONICAL=""` the `case` takes the `"" -> System.version() <> "/"` branch and yields `"1.21.0-dev/"`; the raise only occurs when the var is entirely absent, which the build guarantees against.
- **Trailing-slash asymmetry: search `packages` use `String.trim(canonical, "/")` (line 20) while `deps` interpolate `canonical` verbatim into URLs (lines 86–90).** When `CANONICAL=""` (the normal build path) `canonical` is `"<version>/"`, so `packages` get `"1.21.0-dev"` and `deps` get `".../1.21.0-dev/"` — both correct for their respective consumers (ExDoc package refs want no slash, hexdocs base URLs want the slash). A caller who exports `CANONICAL` without a trailing slash gets a slashless deps URL, but that is the documented convention for the operator-supplied override, not a defect in this file. Basis: `verified` — ran the `canonical` derivation under both `CANONICAL=""` and `CANONICAL="1.19"`; outputs (`"1.21.0-dev/"`/`"1.21.0-dev"`/`".../1.21.0-dev/"` and `"1.19"`/`"1.19"`/`".../1.19"`) match this reading.
- **Remainder of the file (lines 12–252): `extras`, `groups_for_*`, `assets`, `skip_*`, `formatters`, `before_closing_body_tag`.** All static literal data plus one anonymous function that pattern-matches `:html` with a catch-all `_ -> ""` (lines 216–251), so it is total over any atom ExDoc passes. Searched every clause for missing-pattern, precedence, and shape defects; none reach a public entry point since the whole file is a build-time config value. Basis: `asserted` (pure literal data with a total two-clause function; nothing to execute).

## Risks / caveats
The file is a build configuration artifact, not a library module — its "public API" is simply being `Code.eval`'d by the docs tooling with `CANONICAL` set. There is no parser, decoder, or network/untrusted-input surface here, so the bug-class floor from the brief (missing clauses, threading, Unicode, protocols, processes) largely does not apply. The only behavioral coupling worth noting for reviewers of *other* files is the trailing-slash convention on `CANONICAL`: consumers of the generated `deps` URLs assume the operator supplies (or leaves empty, yielding) a trailing slash.

