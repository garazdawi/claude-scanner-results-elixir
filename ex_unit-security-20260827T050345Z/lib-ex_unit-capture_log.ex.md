# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/capture_log.ex
<!-- sha256: 29ab8b3543d6ba8b69de36ed5df43372b471c09a85d329ccdfdfc106bc64a935 -->

_Analysed on 2026-08-27T05:13:35Z_

Both behaviors confirmed: log content is captured verbatim into `StringIO` and returned as an opaque binary (no re-interpretation in this file), the exception path re-raises correctly (`"boom"` propagated, not swallowed), and there is no process leak (proc count 62→62, so `string_io` is closed on the failure path too). My analysis is complete.

## Summary
`ExUnit.CaptureLog` is a ~140-line test helper exposing two documented public functions (`capture_log/2`, `with_log/2`) that mute the default Logger handler, run a caller-supplied 0-arity function, and return the captured log as a binary; every input (`opts`, `fun`, and the log content itself) originates from the trusted test author, no untrusted data reaches any dangerous sink in this file, and the resource-cleanup path is correct — so I found no security defects.

## Security
No findings

### Ruled out
- **`fun.()` invocation (line 127) — arbitrary code execution.** `fun` is the caller's own test function, supplied directly through the public API; it is trusted developer code, not external input. In-contract and by design.
- **`opts` forwarded to `CaptureServer.log_capture_on/3` (line 124) — including `:formatter {module, config}` dispatch.** The `:level`/`:formatter` values are caller-supplied config, not externally derived, and no atom is created from them here. The formatter *module dispatch* happens in `CaptureServer`/`Logger` (out of this file's scope), and the module reference is a literal the caller chose. No atom-table or dispatch-injection surface in this file.
- **Captured log content — potential injection/interpolation sink.** Even when a test captures logs while processing untrusted data (e.g. `Logger.error(attacker_string)`), this file only writes those bytes into `StringIO` and returns them as an opaque binary. Verified: interpolation/shell metacharacters (`#{}`, `$(whoami)`, backticks) are captured verbatim with no re-evaluation. Formatting occurs in the formatter module, not here.
- **`IO.warn/1` deprecation notice (line 115).** Emits a fixed static string with no interpolation of caller data — no format-string/log-injection surface.
- **Resource cleanup `try/after/catch/else` (lines 123–140) — StringIO leak / swallowed exception.** Verified by running: `string_io` is closed on both the success (`else`) and failure (`catch`) paths, capture is disabled in the inner `after`, and exceptions are re-raised via `:erlang.raise/3` (not swallowed). Process count was identical before/after a raising `fun` (62→62), confirming no leak. The residual edge cases (e.g. `Logger.flush/0` raising before `log_capture_off/1` runs) leave only internal `CaptureServer` state and are reachable only if internal callees misbehave — no attacker-controlled path and no security impact.
- **`is_list(opts)` guard (line 112) — dead defensive code check.** Reachable: any non-list `opts` selects no clause and raises `FunctionClauseError`; a non-keyword list reaches `opts[:level]` and raises. Both require a caller violating the documented `capture_log_opts` type spec, which is out of scope, and neither is dead code.
- **Async cross-test log capture.** The moduledoc documents that with `async: true`, messages from concurrent tests may be captured. This is documented, intended behavior within the trusted test-author model, not a defect.

## Risks / caveats
- Scope was limited to this file. The `:formatter {module, config}` value and `opts` are forwarded to `ExUnit.CaptureServer` (which inserts them into a `:public` ETS table and calls `:logger.add_handler/3`); I confirmed no sink exists *in this file*, but the trust handling of those forwarded values lives in `capture_server.ex` and is better judged in a review of that file.
- The reproduction ran against the Elixir toolchain at `/opt/elixir` (built from this source tree) with a manually started `CaptureServer`; a full `mix test` harness was not stood up, but the cleanup/propagation semantics under review do not depend on the surrounding harness.
- All inputs to the two public functions are caller-provided (test author). If a future caller were to derive `opts` (particularly a `:formatter` module) from genuinely untrusted data, the dispatch risk would materialize in `CaptureServer`, not here — worth confirming when that file is reviewed.

