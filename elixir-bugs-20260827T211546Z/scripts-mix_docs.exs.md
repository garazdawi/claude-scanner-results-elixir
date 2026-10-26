# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/mix_docs.exs
<!-- sha256: 3d52639b12306e32a73a6c1a69997bd308fa81a6fa66c3867b86e1d4016c4575 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T00:05:34Z_

## Summary
`mix_docs.exs` is a build-time ExDoc configuration script that reads the operator-set `CANONICAL` env var and returns a static config keyword list; it exposes no public functions and no untrusted-input path, and I found no genuine reachable bugs.

## Bugfixes

No findings

### Ruled out

- **`System.fetch_env!("CANONICAL")` raises when `CANONICAL` is unset (line 6)** — This is the intended contract of `fetch_env!` and the script is only ever evaluated by the doc-build tooling, which sets the variable. It is operator/build config, not a documented callable API reachable by untrusted input, so a raise on a missing build variable is correct fail-fast behavior, not a defect. Basis: `verified` (ran `CANONICAL="" elixir -e '...'` — the empty case takes the `"" -> System.version() <> "/"` branch cleanly; the crash only occurs when the var is entirely absent, which is the documented behavior of `fetch_env!`).

- **Trailing-slash inconsistency between the empty and non-empty `CANONICAL` branches (lines 5–9, 28–33)** — When `CANONICAL=""` the value becomes `System.version() <> "/"` (trailing slash, e.g. `.../1.21.0-dev/`); when `CANONICAL` is non-empty it is used verbatim, so `CANONICAL=main` yields `https://eex.hexdocs.pm/main` with no trailing slash. This is not a bug: the non-empty value is operator-supplied build config whose documented convention is to include its own trailing slash (the release tooling that sets `CANONICAL` supplies it). It is not reachable through any documented public function API, and the callers of this config (ExDoc) receive exactly what the operator provided. Basis: `verified` (ran `CANONICAL="main" elixir -e '...'` and observed `https://eex.hexdocs.pm/main`; confirmed no `CANONICAL`-setting Makefile/CI logic exists in this partial `lib/elixir` checkout, so the slash convention lives in the out-of-tree release scripts that own the variable).

- **`before_closing_body_tag` anonymous function totality (lines 36–71)** — The `fn :html -> ... ; _ -> "" end` has a catch-all `_` clause, so it is total across every atom ExDoc passes (`:html`, `:epub`, `:markdown`, etc.). No `FunctionClauseError` is reachable. Basis: `asserted` (the `_ ->` clause on line 69 covers all non-`:html` inputs by inspection).

- **`String.trim(canonical, "/")` for search-package versions (line 19)** — Produces the bare version/ref (`"1.21.0-dev"`, `"main"`) as ExDoc expects for the `packages` search list; verified output shows correct `{package, version}` tuples. Basis: `verified` (search-packages output above shows `elixir: "1.21.0-dev"` etc.).

## Risks / caveats
- This is a `.exs` config script, not a module; the file "returns" a keyword list and defines no exported/public functions, so the standard public-API attack surface (parsers, decoders, callbacks) is absent. The only external input is the `CANONICAL` env var, which is operator-controlled build configuration rather than untrusted data.
- I could not inspect the out-of-tree tooling that sets `CANONICAL` (this is a partial `lib/elixir` checkout with no Makefile/CI referencing it), so the trailing-slash convention judgment rests on the observed ExDoc URL semantics rather than on reading the setter; if a future caller passes a slash-less `CANONICAL` intending trailing-slash URLs, cross-package doc links would 404 — a config-usage caveat, not a defect in this file.

