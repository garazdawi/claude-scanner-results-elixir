# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/app_loader.ex
<!-- sha256: 6769b1ee50ddfedab9bf9cff9086bcbff8545913b459f9b1f7c054f21f7cd747 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:06:45Z_

## Summary

`app_loader.ex` is Mix's application-cache and `.app`-file loader; it is mostly robust, but `read_cache/1` has a real defect where a cache file whose stored manifest-version does not match the current `@manifest_vsn` escapes the intended graceful fallback and crashes with an uncaught `TryClauseError`.

## Bugfixes

#### `CSCAN-app_loader-read-cache-version-mismatch-crash`
- **Finding ID:** `CSCAN-app_loader-read-cache-version-mismatch-crash`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `app_loader.ex:13-21`, `read_cache/1`
- **Class:** Error-handling asymmetry (`try/else` clause not covered by `rescue`)
- **Impact:** When the on-disk cache holds a *valid* Erlang term whose leading version tag is not the current `@manifest_vsn` — exactly what happens to every existing project after Mix bumps `@manifest_vsn` in an upgrade — `read_cache/1` raises `TryClauseError`. The `else` clause runs *outside* the `try` body, so the `rescue _ -> []` does not catch it. Instead of the intended silent rebuild (return `[]`), the build crashes with an opaque `no try clause matching: {0, %{...}}` until the user manually deletes the manifest or runs `mix clean`. The `@manifest_vsn` tag exists precisely to allow stale caches to be ignored gracefully; this bug defeats that mechanism.
- **Evidence:** `verified` — reproduced the exact control-flow in isolation:
  ```
  $ elixir t.exs
  current vsn: %{a: [1]}
  RAISED: TryClauseError - no try clause matching:
      {0, %{a: [1]}}
  garbage: []
  ```
  A current-version term reads back fine; raw garbage is rescued to `[]`; but a well-formed term with a mismatched version tag raises `TryClauseError`, unrescued.
- **Suggested fix:** Add a catch-all `else` clause so version/shape mismatches degrade to `[]` like every other read failure:
  ```elixir
  else
    {@manifest_vsn, app_to_modules} -> app_to_modules
    _ -> []
  end
  ```

## Risks / caveats

- **`read_app/2` and non-UTF-8 `.app` files:** `String.to_charlist(bin)` assumes the file is UTF-8. An Erlang `.app` file containing latin1 bytes >127 (e.g. in a `description` string) would raise `UnicodeConversionError` rather than returning `:invalid`. `.app` files are conventionally ASCII, so I did not classify this as a confirmed reachable bug, but a dependency shipping a latin1-encoded `.app` would crash instead of being reported invalid. Not verified against a real-world latin1 `.app`.
- `stale_cache/1`, `write_cache/2`, `load_apps/5`, `traverse_apps/5` depend on `Mix.Project`/`Mix.Dep`/`Mix.State` global state that I could not stand up in isolation; I reasoned about their control flow (accumulator threading, nil `ebin_path` handling, `app_children/1` being reached only after the app is loaded) and found no defect, but did not execute them end-to-end.
- The severity of the confirmed finding depends on a future/historical `@manifest_vsn` change producing a mismatched-but-valid on-disk term; the crash mechanism itself is verified, the trigger frequency is an upgrade-time event.

### Ruled out
- **`load_app/2` returning `:invalid`/`:missing` through the bare `with`:** matches `read_app/2`'s documented return surface (`{:ok, _} | :invalid | :missing`); callers in `traverse_apps` only test for `{:ok, _}`, so the pass-through is intentional, not a wrong shape.
- **`app_children/1` on an unloaded app (`nil ++ nil`):** reached only after `Application.spec(app, :vsn)` is truthy or `load_app` succeeded (which loads the app), so `Application.spec(app, :applications)` returns a list, not `nil`; no crash path found.

