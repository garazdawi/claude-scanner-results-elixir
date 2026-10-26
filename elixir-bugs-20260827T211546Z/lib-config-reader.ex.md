# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/config/reader.ex
<!-- sha256: 86a171aff0d1fe31f127fd2942a7ca6c8010b1931116b19c3abe997ba0a499c9 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/config/reader.ex
<!-- sha256: 86a171aff0d1fe31f127fd2942a7ca6c8010b1931116b19c3abe997ba0a499c9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (hedge-clearing pass). Delegation targets read at their real install paths (`/opt/elixir/lib/elixir/lib/config.ex`, `/opt/elixir/lib/elixir/lib/config/provider.ex`); every guard, projection, and doctest re-executed this pass against Elixir 1.21.0-dev / OTP 29._

Both guards behave as documented. This file is a thin, correct delegation layer. The first-pass hedges ("outside this file") are cleared: the delegation targets were read this pass, and each externally-observable shape was re-executed (not merely re-asserted), so every axis on every ruled-out entry was actually tried.

## Summary

`Config.Reader` is a thin wrapper that delegates all substantive work (evaluation, merging, path resolution) to `Config`/`Config.Provider`. Its own logic — clause dispatch in `init/1`, the `:imports`-disabled guard in `read_imports!/2`, the `elem(0)` / pass-through projections in `eval!/read!/read_imports!`, and the `resolve → read! → merge` chain in `load/2` — is correct and matches its docs/doctests. Reading the neighbour source (`Config.__eval__!` at `config.ex:318-343`, `Config.__merge__` at `config.ex:368-381`, `Config.Provider.validate_config_path!` at `provider.ex:172-191`, `resolve_config_path!` at `provider.ex:197-199`) told me what those guards *do*; re-running the file's own guards and projections through them told me what they *return*, which is what actually clears the axes below.

## Bugfixes

No findings.

### Ruled out

- **`init/1` clause ordering (`is_list(opts)` first, then `init(path)`)** — A keyword-list argument matches the first clause; a non-list path (binary or `{:system,…}` tuple) falls to the second, which re-enters as `init(path: path)`. Executed this pass: `init("/etc/config.exs") → {"/etc/config.exs", []}`; `init({:system,"RELEASE_ROOT","/c.exs"}) → {{:system,"RELEASE_ROOT","/c.exs"}, []}`; `init(path: "/x.exs", env: :prod) → {"/x.exs", [env: :prod]}`. Charlist axis executed: `init(~c"/etc/config.exs")` matches `is_list/1`, then **raises `%KeyError{key: :path}`** in `Keyword.pop!` — it does *not* silently succeed as a path (a charlist is a list of integers, not a keyword list, so `:path` is absent). `Path.t()` is `binary()`, so charlist input is spec-violating misuse and the loud `KeyError` is acceptable. **New axis executed this pass — relative binary:** `init("relative/path.exs")` reaches `validate_config_path!/1` (`provider.ex:177`) which raises `%ArgumentError{}` because `Path.type/1 == :relative` — the provider contract rejects non-absolute paths at `init` time, not `load` time. Basis: `verified`. Axes examined: keyword-list, absolute-binary, `{:system,…}` tuple, charlist, relative-binary, empty-list | Not examined: none.
- **`init/1` with a keyword list missing `:path`** — Raises `KeyError` via `Keyword.pop!`; `:path` is documented required, so raising is the contract. Executed: `init(env: :prod)` raises `%KeyError{key: :path, term: [env: :prod]}`; `init([])` raises `%KeyError{key: :path, term: []}`. Basis: `verified`. Axes examined: missing-key non-empty, empty-list | Not examined: none.
- **`read_imports!/2` with `imports: :disabled`** — Documented to reject disabling imports; the local `if opts[:imports] == :disabled` guard (`reader.ex:115`) raises `ArgumentError` before any delegation. Executed: `read_imports!("cfg.exs", imports: :disabled)` raises `%ArgumentError{message: ":imports must be a list of paths"}`; control case `read_imports!("cfg.exs", [])` returns `{[app: [k: :v]], ["/tmp/cfg.exs"]}`. Basis: `verified`. Axes examined: `:disabled` sentinel, empty-opts control | Not examined: none.
- **`merge/2` doctests** — All three published doctests reproduced exactly, executed: `merge([app: [k: :v1]], [app: [k: :v2]]) → [app: [k: :v2]]`; `merge([app: [k: [v1: 1, v2: 2]]], [app: [k: [v2: :a, v3: :b]]]) → [app: [k: [v1: 1, v2: :a, v3: :b]]]`; `merge([app1: []], [app2: []]) → [app1: [], app2: []]`. Deep-merge lives in `Config.__merge__` (`config.ex:369-381`), read this pass: `Keyword.merge/3` at both app and key level, descending only when *both* values satisfy `Keyword.keyword?/1`, else `value2` wins — consistent with the executed doctests (scalar override, nested-keyword deep-merge, disjoint-app union). Basis: `verified`. Axes examined: scalar override, nested-keyword deep-merge, disjoint-app union | Not examined: none.
- **`eval!/2` and `read!/2` return shape (`|> elem(0)` → keyword)** — **Upgraded from hedged assumption in first pass, re-confirmed by execution this pass.** Read `Config.__eval__!` (`config.ex:318-343`): `@spec {keyword, [Path.t()] | :disabled}`; both return arms yield `{config, imports}` with config as element 0. Executed: `Config.__eval__!(Path.expand("cfg.exs"), "import Config\nconfig :app, k: :v", []) → {[app: [k: :v]], ["/tmp/cfg.exs"]}`; `eval!("dummy.exs", …) → [app: [k: :v]]`; `read!("cfg.exs") → [app: [k: :v]]`. `read_imports!/2` deliberately returns the full tuple: executed `{[app: [k: :v]], ["/tmp/cfg.exs"]}`, matching `@spec {keyword, [Path.t()]}`. All three projections in this file (`elem(0)`, `elem(0)`, tuple pass-through) match the executed shapes. Basis: `verified`. Axes examined: `elem(0)` on eval, `elem(0)` on read, tuple pass-through on read_imports | Not examined: none.
- **`load/2` path resolution + merge chain** — Read `Config.Provider.resolve_config_path!/1` (`provider.ex:197-199`): binary passes through unchanged; `{:system, name, path}` becomes `System.fetch_env!(name) <> path`. `load/2` wires `resolve → read! → merge(config, …)`. Executed both path axes this pass: binary — `load([app: [k: :old]], {"cfg.exs", []}) → [app: [k: :v]]` (second config overrode first, confirming `merge(config, read!)` argument order); `{:system,…}` — with `CFGROOT=/tmp/`, `load([], {{:system, "CFGROOT", "cfg.exs"}, []}) → [app: [k: :v]]` (resolved to `/tmp/cfg.exs` and read). Relative/non-binary rejection is enforced upstream at `init` (see first entry) via `validate_config_path!`; `resolve_config_path!` itself has no relative-guard, consistent with the contract that `init` already validated. Basis: `verified`. Axes examined: binary path, `{:system,…}` env-resolved path, merge argument-order | Not examined: none.

## Risks / caveats

The *internal* evaluation semantics of `Config.__eval__!` — process-dictionary threading of opts/config/imports (`put_opts`/`put_config`/`put_imports`), `Code.eval_string` execution of arbitrary config source, and the recursive-import / `:disabled` guards in `import_config!` (`config.ex:328-366`) — are the property of `config.ex` and belong to that file's own review. This file neither branches on nor reshapes those internals; it only projects element 0 (or passes the tuple through). Reading `config.ex` this pass confirmed the return shape but did not audit that evaluation machinery, which is out of scope for `reader.ex`.

## Did I verify this?

Yes, with executed evidence produced in this pass (not carried over). I read the three delegation targets at their real install paths and re-ran every guard, projection, and doctest of `reader.ex`:

- `init/1` string / `{:system,…}` / keyword / charlist / relative-binary / empty-list / missing-`:path` dispatch — executed; charlist and relative-binary are the sharpened axes.
- `read_imports!` `:disabled` → `ArgumentError`, empty-opts control → tuple — executed.
- `eval!/read!/read_imports!` return shapes and raw `Config.__eval__!` tuple — executed, `{keyword, imports}`.
- All three `merge/2` doctests — executed, exact.
- `load/2` binary and env-resolved `{:system,…}` wiring incl. merge argument order — executed.

Every axis on every ruled-out entry was actually tried this pass; no entry carries a residual `Not examined` axis. Reading the neighbour modules is cited only for *what* the guards do — the axis clearances rest on the executed runs above.

## Dropped in re-review

None. No prior finding existed to drop. No ruled-out entry was moved to a finding: every hedge cleared to `verified` via an executed run this pass (the safe direction only because it is backed by an actual run, not a re-read). One axis (relative-binary) was newly tried and added to the `init/1` entry, further narrowing — not widening — its `Not examined` list to `none`.
