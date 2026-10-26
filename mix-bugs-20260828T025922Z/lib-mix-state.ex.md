# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/state.ex
<!-- sha256: 6fc72b6cf2a4cb813714aa50dc6c1086c3b6418d4b0d899a740d31881b419747 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:28:00Z_

## Summary

`Mix.State` is a small internal (`@moduledoc false`) state holder — an ETS-backed key/value store plus a `persistent_term` cache and a lazily-built map of builtin OTP apps — and I found no genuine, reachable behavioural defects in it.

## Bugfixes

No findings.

## Risks / caveats

- The module is `@moduledoc false`, so there is no formally *documented* public API; I treated the exported functions (`builtin_apps/0`, `fetch/1`, `get/2`, `put/2`, `update/2`, and the `*_cache` functions) as the effective contract that Mix callers rely on, since the scan target is the file itself.
- All positive checks were run against the source-built toolchain (`/opt/elixir`, `/usr/local/lib/erlang`): `Mix.State.get/fetch/put/update` behave as expected, `builtin_apps/0` returned 41 apps with no `nil` keys, and version-suffixed dirs resolve correctly (`crypto-5.9 → :crypto`, `../lib/elixir/ebin → :elixir`).
- I did not stress `app_from_code_path/1` against exotic code-path strings from `:code.get_path()` in a non-standard OTP layout, but the parser degrades to `nil` (and is filtered out) on any path not ending in `.../ebin`, so a mis-shaped entry yields omission, not a crash.

### Ruled out

- **`app = app_from_code_path(path)` leaking `nil` keys into the map** — verified that an assignment qualifier in an Elixir `for` comprehension filters falsy results: `for x <- [1, nil, 2], y = (if x, do: x*10), do: {x,y}` returned `[{1,10},{2,20}]`, and `builtin_apps/0` produced no `nil` keys. Not a bug.
- **`collect_dir/2` resetting the accumulator on `-`** — this correctly strips the `name-vsn` version suffix; it only mishandles multi-hyphen app-dir names, which OTP/Elixir builtin apps never have, and `collect_dir` stops at the first `/`/`\` so hyphens in parent directories are irrelevant. Verified `crypto-5.9 → :crypto`. Not reachable in practice.
- **`update/2` raising `ArgumentError` on a missing key** — verified it raises (via `:ets.lookup_element/3`), but this is only called internally with keys seeded in `init/1`; no documented contract promises an error tuple, and raising on an absent key is defensible. Not an in-scope defect.
- **`from_env/1` atom creation via `String.to_unsafe_atom`** — private, invoked only from `init/1` on operator-controlled `MIX_ENV`/`MIX_TARGET`; empty/absent values fall back to the default (verified empty `MIX_ENV` → `:dev`). Atom-table growth is operator-bounded and flagged as intentional in the threat model, not a network-reachable defect.
- **`builtin_apps` cache coherence** — the list→map upgrade happens inside the serialized `handle_call`, so no race; second call returns the cached map. Not a bug.

