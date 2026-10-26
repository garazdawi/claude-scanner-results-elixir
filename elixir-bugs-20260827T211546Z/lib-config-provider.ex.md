# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/config/provider.ex
<!-- sha256: 51b99db9cb90130e2c48b766722a74de24344e954ffd6fba0fedda6a6aa89152 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:57:57Z_

## Summary

`Config.Provider`'s documented public surface (`validate_config_path!/1`, `resolve_config_path!/1`, and the `init/1`/`load/2` behaviour callbacks) behaves as specified, and no reachable bug was found on those paths.

## Bugfixes

No findings

### Ruled out

- **`validate_config_path!/1` returning wrong result or crashing on documented inputs** — Ran all three documented shapes: `{:system, bin, bin}` → `:ok`, absolute binary → `:ok`, relative binary → `ArgumentError`. Non-binary/non-tuple falls to the catch-all and raises `ArgumentError` with a descriptive message (the bang-function contract). The `and` on line 178 short-circuits so `Path.type/1` is never called on a non-binary. Basis: verified.
- **`resolve_config_path!/1` producing a non-binary or mis-concatenating** — Ran `{:system,"MYVAR","/sub"}` with `MYVAR=/root` → `"/root/sub"`, and `"/plain"` → `"/plain"`. Matches the `@spec ... :: binary` and the doc's "concatenation of the environment variable with the given path". `System.fetch_env!/1` raising when the env var is unset is the documented `fetch_env!` behaviour, not a defect. Basis: verified.
- **`maybe_validate_compile_env/1` mis-handling the default `false`** — `provider.validate_compile_env` defaults to `false`; the `with [_ | _] = compile_env <- false` clause fails to match and the `with` returns `false` without calling `validate_compile_env/2` (no `else` needed since the return value is discarded). Only a non-empty list proceeds to validation. Basis: asserted (read of lines 284–289 with the struct default on line 163).
- **`read_config!/1` `CaseClauseError` on an empty or multi-term config file** (lines 391–400 handle only `{:ok, [inner]}` and `{:error, _}`) — Not reachable through the documented public API: `read_config!` is called only from `boot/1` (`@doc false`), and the file it consults is the machine-generated runtime `sys.config` written by `write_config!/2` (always exactly one term). No documented entry point lets untrusted data reach it with zero or multiple terms. Basis: asserted (call-graph is local to this file: `read_config!` ← `boot_providers` ← `boot`, all non-public).
- **`traverse_env`/`valid_compile_env?`/`validate_compile_env` clause gaps (e.g. empty `path` list → `FunctionClauseError` on `[key | path]`)** — All three are `@doc false` and consume the internal `validate_compile_env` state built by `init/3`; reaching a bad shape requires violating the internal contract, which is out of scope. Basis: asserted (`@doc false` on lines 291/302; inputs originate from `init/3`'s `validate_compile_env` option).
- **`run_providers/1` swallowing or mis-checking provider return values** (lines 403–420) — Non-list returns are aborted with a clear message; thrown/raised/exited errors are re-raised after logging via `:erlang.raise/3` preserving kind and stacktrace. Symmetric across `error`/`throw`/`exit`. Basis: asserted (read of the `try/catch/else`).

## Risks / caveats

- Review was confined to `/Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/config/provider.ex`; the deep-merge and file-format handling live in `Config`/`Config.Reader` and were not examined.
- Several boot-path functions (`boot/1`, `boot_providers/2`, `run_providers/1`, `read_config!/1`, `write_config!/2`, `validate_compile_env/2`) are `@doc false` and reachable only through the release boot machinery, not a documented public entry point, so per the stated scope they were treated as internal; a defect confined to those paths would fall outside this review's mandate.

