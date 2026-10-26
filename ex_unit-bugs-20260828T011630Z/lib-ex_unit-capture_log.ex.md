# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/capture_log.ex
<!-- sha256: 29ab8b3543d6ba8b69de36ed5df43372b471c09a85d329ccdfdfc106bc64a935 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/capture_log.ex
<!-- sha256: 29ab8b3543d6ba8b69de36ed5df43372b471c09a85d329ccdfdfc106bc64a935 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`ExUnit.CaptureLog` is correct: both documented entry points (`capture_log/2`, `with_log/2`) return the documented shapes, filter by level, honor the `:warn`→`:warning` deprecation, and close the `StringIO` device on both the success and exception paths. In this re-review pass I eliminated the two "out of scope / single-file" hedges from the first pass by reading the sole callgraph neighbour (`ExUnit.CaptureServer`) and by executing the delegated `:formatter` / extra-opts contract against the real modules under Erlang/OTP 29. No reachable defect was found.

## Bugfixes

No findings

### Ruled out

- **Resource leak of the `StringIO` device on the exception path** — The outer `try` closes `string_io` in *both* the `catch` clause (line 134) and the `else` clause (line 138), and the inner `after` (lines 128–131) turns capture off before the exception re-raises. Executed: `with_log(fn -> raise "kaboom" end)` re-raises `"kaboom"` unchanged and the process count delta before/after (measured after a 50 ms settle) is `0` — no leaked `StringIO`/`Task` process. Confidence: High. Axes examined: success path, raise path, process-leak count. Not examined: none.

- **`log_capture_on` raising leaves an open device** — If `ExUnit.CaptureServer.log_capture_on/3` raises before the inner `try` is entered, control reaches the outer `catch`, which closes `string_io`. `ref` is unbound at that point but is never referenced in the `catch` clause, so no secondary crash. Confirmed by reading lines 123–135 here plus `capture_server.ex:73–99`: `log_capture_on` is a `GenServer.call`; the only raise before the inner `try` is a `call` failure (e.g. server not started), which exits and is caught, closing the device. Confidence: High. Axes examined: server-not-started exit path, `ref`-unbound reference in catch. Not examined: none.

- **`:warn` deprecation branch mis-handling** — `opts[:level] == :warn` emits the `IO.warn` deprecation and rewrites the level via `Keyword.put(opts, :level, :warning)`; capture still proceeds. Verified previously: `capture_log([level: :warn], fn -> Logger.warning("w") end) =~ "w"` returns `true` and the deprecation warning fires exactly once. Confidence: High. Axes examined: level rewrite, capture continuity, single-emission of deprecation. Not examined: none.

- **`:level` filtering wrong direction / off-by-one** — Verified `capture_log([level: :error], …)` suppresses an `info` message but keeps an `error` message (`{false, true}`), matching the documented "lower level will be ignored" contract. This is enforced downstream in `capture_server.ex:243` via `:logger.compare_levels(event.level, level) in [:gt, :eq]`, consistent with the direction observed. Confidence: High. Axes examined: below-threshold suppression, at-threshold retention. Not examined: none.

- **Default-argument dispatch (`opts \\ []`) sending `fun` to the wrong parameter** — `capture_log(fn -> … end)` correctly binds the single argument to `fun` with `opts = []`; verified `capture_log/1` returns the captured string. Same generated-clause behavior applies to `with_log/2` (the `when is_list(opts)` guard on line 112 rejects a lone `fun` from being taken as `opts`). Confidence: High. Axes examined: arity-1 dispatch, `is_list` guard. Not examined: none.

- **`@spec`/return-shape mismatch** — Executed: `with_log(fn -> Logger.error("x"); 4 end)` returned `{4, "…[error] x\n"}` (a `{result, String.t()}` tuple) and `capture_log/2` returned the bare `String.t()`, both matching their `@spec` declarations. Confidence: High. Axes examined: `with_log` tuple shape, `capture_log` bare-string shape. Not examined: none.

- **`:formatter` / extra-opts forwarding contract (delegated to `CaptureServer`)** — *Upgraded from hedged caveat in first pass* (was "out of scope for this single-file review"), now **ruled out**. Read `capture_server.ex:73–99`: `log_capture_on` does `{level, opts} = Keyword.pop(opts, :level)` then `Keyword.get_lazy(opts, :formatter, fn -> Logger.default_formatter(opts) end)`, so a supplied `{module, config}` formatter is used verbatim and, when absent, the remaining opts are forwarded to `Logger.default_formatter/1` as documented. Executed against the real modules: (1) `[formatter: {MyFmt, :mycfg}]` produced `"FMT<error>:mycfg\n"` (custom module + config both honored); (2) `[format: "$message\n"]` produced `"plain\n"` (extra opt forwarded as a default-formatter override); (3) baseline with no opts used `Logger.default_formatter/1`. All three match the `@doc` on lines 51–82. Confidence: High. Axes examined: custom-formatter delegation, extra-opt forwarding, default fallback. Not examined: none.

## Risks / caveats

- `StringIO.open/1` starts a process that is not linked to the caller; if the *calling* process is killed with `:kill` between `StringIO.open` (line 121) and either close site, that device would leak. This is not reachable through the documented API by ordinary means (no in-band way to brutal-kill mid-call), so it is not a bug in this file — it is the one theoretical leak window. Confidence: Low (theoretical, not reachable via documented API).

## Did I verify this?

Yes for every ruled-out entry above. Beyond re-reading `capture_log.ex`, I read the sole callgraph neighbour it invokes, `ExUnit.CaptureServer` (`log_capture_on/log_capture_off`, `log/2`, `remove_log_capture`, `maybe_revert_to_default_handler`), and I executed a script under Erlang/OTP 29 / Elixir 1.21 that drove `capture_log/1,2` and `with_log/2` through: a custom `:formatter`, an extra formatter override opt, the default formatter, the `{result, log}` return shape, and the raise path with a before/after process-count leak check. Reading the neighbour resolved the *structure* of the delegation; the executed run resolved the *behavior*, which is why the formatter caveat could be moved to `verified` rather than left `asserted`.

## Dropped in re-review

- **"Behavior of the `:formatter` option … delegated entirely to `ExUnit.CaptureServer` / `Logger`; out of scope for this single-file review"** — dropped as a hedged caveat and re-entered as a verified "Ruled out" item (see the `:formatter` / extra-opts entry above). New evidence: `capture_server.ex:73–99` shows the pop/`get_lazy` delegation, and three executed captures confirmed custom-formatter, extra-opt-forwarding, and default-fallback all behave as documented. The hedge is gone; nothing about it now rests on an unverified assumption.

The theoretical `:kill`-between-open-and-close window is retained (not dropped) as an explicit Low-confidence caveat, since it remains unreachable through the documented API and I have no executed evidence of a reachable leak.
