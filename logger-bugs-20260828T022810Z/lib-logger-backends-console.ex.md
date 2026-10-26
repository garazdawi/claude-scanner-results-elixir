# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/console.ex
<!-- sha256: 217443a1bd0e306bdcebff542d8725de83755b82601199f375c9a3aabab4ff10 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:37:05Z_

## Summary

Reviewing `Logger.Backends.Console` (the deprecated `:gen_event` console backend) end-to-end against its documented public API and gen_event callback contracts turned up no bug triggerable through documented usage; the one crash I could force requires internal-metadata misuse the real logging pipeline never produces.

## Bugfixes

No findings.

### Ruled out

- **`color_event` `Map.fetch!(colors, level)` KeyError on `:warn`** (`console.ex:252`). `color_event` looks the level up in a colors map keyed by the eight syslog levels (`emergency…debug`), with no `:warn` key. I confirmed by construction that feeding `handle_event/2` a `:warn`-level event with colors enabled and *no* `erl_level` metadata raises `KeyError` (verified: `RAISED: key :warn not found in: %{error: :red, ...}`). However, this shape is not producible via the documented path. Every event reaching this backend comes from either `Logger.Backends.Handler.log/2`, which always injects `[erl_level: erl_level | ...]` metadata carrying the *original* erlang level (`handler.ex:58`), or `Logger.Backends.Config.warn/1`, which emits a `:warning`-level event directly (`config.ex:102-103`). Line 68 (`{:erl_level, level} = List.keyfind(md, :erl_level, 0, {:erl_level, level})`) then overrides `level` to that erlang level (`:warning`, `:notice`, `:emergency`, …), all of which *are* keys in the colors map. Verified the real path is safe: an event `{:warn, :gl, {Logger, "hi", ts, [erl_level: :warning, pid: …]}}` produced `"\e[33mhi\n\e[0m"` with no crash. Reaching the `:warn` KeyError requires a caller to log while suppressing/overwriting the internal `erl_level` metadata key — misuse of an undocumented internal field, out of scope.
- **`meet_level?` / `Logger.compare_levels` on erlang-style levels** (`console.ex:121-122`). After the line-68 override, `level` can be `:warning`/`:notice`/`:emergency`/etc. Verified `Logger.compare_levels/2` accepts all eight (`{:emergency,:gt} … {:debug,:lt}`), so no `FunctionClauseError`.
- **`retry_log` `else _ -> raise` on well-formed data** (`console.ex:298-300`). When `:unicode.characters_to_binary(dirty)` returns a plain binary (valid data), the `_` clause raises. This is intentional: `retry_log` is only entered after the IO device *already* reported an error, and a device erroring on good data is unrecoverable by design (the comment states this). The malformed case returns `{_, good, bad}` and is pruned/retried. Not a defect.
- **`handle_io_reply` totality** (`console.ex:263-282`). Clauses cover `:ok` and every `{:error, _}` shape with a catch-all; a non-`:ok`/non-`{:error,_}` reply would fall through, but that violates the Erlang IO protocol contract the device must honor. Not reachable via documented input.
- **Monitor leak on error paths.** `async_io` monitors the device (`console.ex:207`); the `:ok` reply (`handle_io_reply/2`, line 264) and every `retry_log` (line 285) demonitor with `[:flush]` before creating a new monitor, and the DOWN/unknown-error paths crash the handler (which tears down its monitors). Balanced; no leak on documented paths.
- **Buffer/backpressure state threading** (`handle_event` cond, `buffer_event`, `log_buffer`, `await_io`, `flush`). Buffering only occurs while an async IO is in flight (`ref` non-nil), the full-buffer branch blocks in `await_io`, and `flush/1` recurses until `ref` is `nil`; `output` always tracks the in-flight request (single via `log_event`, batch via `log_buffer`). No stale-accumulator or non-termination issue found.
- **`take_metadata` ordering.** `configure_metadata/1` reverses the key list and `take_metadata/2` prepends during `reduce`, restoring original key order — deliberate double-reversal, not a bug.

## Risks / caveats

- I did not exercise a fully-booted `Logger` application with this backend registered under its supervisor; findings rely on driving the exported `:gen_event` callbacks directly and tracing the two real event producers (`handler.ex`, `config.ex`). If a future/alternate producer emits a `{level, gl, {Logger, …}}` event whose collapsed `level` is `:warn` (or any non-syslog atom) *without* an `erl_level` metadata key, the ruled-out `Map.fetch!` KeyError at `console.ex:252` becomes live — the safety of that line is an invariant of the upstream producers, not of this file.
- The module is deprecated; behavior is judged against its documented backwards-compatibility contract, not the current `:default_handler`.

