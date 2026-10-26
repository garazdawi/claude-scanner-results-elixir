# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger.ex
<!-- sha256: e203c4f4541eb33a91f6dd2610f6b59f42016468cf0c28c6a038f98843a3d2bb -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:33:40Z_

## Summary

The file is a faithful implementation of Elixir's `Logger` public API; its documented entry points, doctest, and metadata/level handling all behave correctly under test, and the only defect found is a cosmetic documentation typo.

## Bugfixes

#### `CSCAN-logger-warn-maps-to-warnings-typo`
- **Finding ID** — `CSCAN-logger-warn-maps-to-warnings-typo`
- **Severity** — Low
- **Confidence** — high
- **Location** — `logger.ex:499`, module `@moduledoc` ("Backends and backwards compatibility" section)
- **Class** — Documentation mismatch
- **Issue** — The moduledoc states: "`Logger.Backends` only receive `:debug`, `:info`, `:warning`, and `:error` messages. `:notice` maps to `:info`. `:warn` maps to `:warnings`. All others map to `:error`." The target `:warnings` does not exist — it is not one of the eight `Logger` levels (`@levels`/`level()`), nor is it among the four backend levels the very same sentence enumerates (`:debug`, `:info`, `:warning`, `:error`). The intended target is `:warning`.
- **Suggested fix** — Change `:warn` maps to `:warnings` to `:warn` maps to `:warning`.
- **Impact / threat** — Purely informational: a reader following the backend level-mapping description is told a deprecated alias maps to a non-existent level. No runtime path consumes this text, so no caller is broken; the cost is reader confusion only. No untrusted-data exposure.
- **Evidence** — `asserted`. The contradiction is internal to the sentence (`:warnings` is absent from the four valid targets it lists and from `@levels` at line 533), so no execution is needed; the mapping itself lives in `Logger.Backends` (out of this file) and was not exercised.

### Ruled out
- **`compare_levels/2` doctest** — Verified by running: `Logger.compare_levels(:debug, :warning)` returns `:lt` and `Logger.compare_levels(:error, :info)` returns `:gt`, exactly matching the two doctest lines (lines 705–708).
- **`metadata/1` nil-deletion / `:undefined` branch** — Verified: setting `b: nil` removes key `b`, `reset_metadata/1` filters out nil values, and the `:undefined` path routes through `reset_metadata`. All returned `:ok`-consistent results and correct maps.
- **`configure/1` deprecation on `:truncate`/`:utc_log`** — These keys appear in both `@valid_options` and `@backend_options`, so setting them emits a "deprecated" warning despite being listed in the `configure_opts` type. Confirmed by running, but this matches upstream Elixir behavior (backends are deprecated) and is intended, not a defect.
- **`bare_log/3` with `:all`/`:none`** — Would raise `FunctionClauseError` in `__do_log__` since those meta-levels are not in `@levels`, but the `@spec` restricts `level` to the eight real levels, so this requires violating the documented contract (out of scope).
- **`__do_log__` non-string/list/map message** — The deprecation branch calls `to_string/1`, which can raise `Protocol.UndefinedError` for e.g. a tuple, but only on inputs the emitted deprecation message and `message` type explicitly disallow (contract misuse).
- **`get_process_level`/`put_process_level`/`enable`/`disable`/`enabled?`** — Verified round-trip through `self()`: store/read of the process level works and matches docs (`nil` when unset). The `pid == self()` guard is documented ("Currently the only accepted PID is `self()`").
- **Level macros / `log/3` runtime level / anonymous-function message with metadata tuple** — Verified end-to-end: `info`, `debug` (iodata), `log/3` with a `fn -> {msg, meta} end`, runtime-variable level, and `warn` deprecation all produce correct level-tagged output; metadata merge order (invocation overrides existing) is correct.

## Risks / caveats
- `/opt/elixir` is built from the source under review, so behavioral checks were run against this exact code; the doctest and API smoke tests above passed.
- Several public functions (`put_module_level`, `get_module_level`, `add_handlers`, etc.) are thin `defdelegate`s to Erlang's `:logger`; their correctness depends on OTP and was not audited here — only the delegation targets (`set_module_level`, `unset_module_level`, etc.) were confirmed to name the right functions.
- The one finding is documentation-only; I found no functional defect reachable through the documented public API of this file.

