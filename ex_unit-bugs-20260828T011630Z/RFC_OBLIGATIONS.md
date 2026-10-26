<!-- rfc-governed: no -->
<!-- standards: -->

# Documented-Contract Review List — `elixir/lib/ex_unit`

This folder is the source of **ExUnit**, Elixir's built-in unit-testing framework (`ExUnit`, `ExUnit.Case`, `ExUnit.Assertions`, `ExUnit.Callbacks`, `ExUnit.Runner`, `ExUnit.DocTest`, `ExUnit.Diff`, `ExUnit.CaptureIO/CaptureLog`, `ExUnit.CaptureServer`, formatters, `FailuresManifest`, `Filters`, `Server`, `EventManager`, `OnExitHandler`). It is pure runtime/library/OTP code: it defines and runs test cases, manages setup/teardown callbacks, captures IO/log output, computes diffs, and formats results to the console. It speaks **no** network protocol, implements **no** wire format, and runs **no** standardized cryptographic algorithm.

I inspected the formatters and the whole `lib/` tree for any interoperable external report format (JUnit/Surefire XML, JSON schema, vendor wire format). ExUnit's bundled formatters (`ExUnit.CLIFormatter`, `ExUnit.Formatter`) emit **human-readable console output only** — there is no XML/JUnit/Surefire report generation here (that lives in the third-party `junit_formatter` package, not in this tree). The only "protocol" references found are Elixir *protocols* (`String.Chars`, the formatter protocols) and a `:rand` algorithm mention — OTP/Elixir-internal, not external standards.

## Standards this repository references
The scan reported **no standards references found in the application tree**, and my reading confirms it: nothing in this folder implements an IETF RFC, NIST/FIPS publication, ITU-T/IEEE protocol spec, W3C/ISO document format, or a de-facto external report schema. There is therefore no external standard to account for, and the RFC/algorithm-compliance pass has nothing to adjudicate.

## Applicable standards / contracts

### External standards
None. (The `<!-- standards: -->` marker is intentionally empty.) The downstream compliance pass has nothing to adjudicate for this folder.

### OTP-internal contracts (per-file context, NOT compliance obligations)
These are guidance for the per-file reviewer only — they are *not* standard-conformance obligations:

- **Behaviour / callback shapes** — `ExUnit.Formatter` consumers implement a `GenServer`-style event handler (`init/1`, `handle_cast/2`); `ExUnit.CaseTemplate` and `ExUnit.Callbacks` define `setup`/`setup_all`/`on_exit` contracts whose return values (`:ok`, `{:ok, context}`, `%{...}` context merge) are documented and must be respected.
- **`-spec`/`@doc` return shapes** — public assertion/capture/filter functions document their success and failure return tuples (`{:ok, _}` / `{:error, _}` / raising `ExUnit.AssertionError`).
- **Setup/teardown ordering and context propagation** — documented `@doc` contracts in `callbacks.ex`.

## Checkable sub-claims

1. **RESOURCE_CLEANED_ON_ERROR_PATH** — Captured resources (IO/log capture devices in `capture_io.ex`, `capture_log.ex`, `capture_server.ex`; monitored processes, ETS/registered groups in `server.ex`, `on_exit_handler.ex`) must be released on every error/exception exit, not just the success path (`try/after`). (OTP resource-cleanup convention.)
2. **ON_EXIT_RUNS_ON_FAILURE** — `on_exit` / teardown callbacks registered in `callbacks.ex` / `on_exit_handler.ex` must still execute when the test body raises or exits, and their own failure must be reported without swallowing the original test failure. (`ExUnit.Callbacks` `@doc`.)
3. **DOCUMENTED_RETURN_SHAPE** — Public functions must return the tuple/struct shape their `@spec`/`@doc` declares (e.g. assertions raise `ExUnit.AssertionError`; capture functions return the captured value/string), with no silent demotion. (`-spec`/`@doc`.)
4. **HANDLE_CAST_TOLERATES_UNKNOWN** — Event-manager / formatter / server `handle_cast`/`handle_info`/`handle_call` callbacks (`event_manager.ex`, `server.ex`, `runner_stats.ex`, `cli_formatter.ex`) must not crash on an unexpected message; unknown events should be ignored, not `badmatch`. (`gen_server(3)` / `gen_event` convention.)
5. **ERROR_PROPAGATES_NOT_BADMATCHES** — A failed internal call (compile of a doctest in `doc_test.ex`, filter parse in `filters.ex`, manifest read in `failures_manifest.ex`) must surface as a documented error/exception, not an uncaught `badmatch` deep in the stack. (OTP error-handling convention.)
6. **MANIFEST_CORRUPTION_TOLERATED** — `failures_manifest.ex` reading a missing / truncated / version-mismatched manifest file must degrade gracefully (treat as empty / stale) rather than crash the run. (`@doc` contract for persistence.)
7. **CAPTURE_RESTORES_GROUP_LEADER** — `capture_io`/`capture_log` must restore the original group leader / logger backend after capture, even when the captured block raises. (Capture `@doc` contract.)
8. **TIMEOUT_ENFORCED** — Test / setup timeouts configured in `runner.ex` / `callbacks.ex` must actually terminate the runaway test process and report a timeout, not hang indefinitely. (Documented `:timeout` option.)
9. **SEED_DETERMINISM** — Configured `:seed` / `:rand` algorithm in `ex_unit.ex` must produce a reproducible test order for a given seed. (Documented `@doc` behaviour.)
10. **ASSERT_MESSAGE_NO_EVAL_SIDE_EFFECTS** — Assertion macros in `assertions.ex` must evaluate the asserted expression exactly once (no double evaluation of side-effecting expressions when building the failure message). (`ExUnit.Assertions` `@doc`.)
11. **FILTER_PARSE_INVALID_RETURNS_ERROR** — `filters.ex` must reject malformed `--only`/`--exclude` filter expressions with a documented error rather than silently matching everything/nothing. (`@doc`.)
12. **DIFF_TERMINATES** — `diff.ex` structural/collection diffing must terminate and not blow the stack/heap on large or deeply nested/cyclic terms. (Internal robustness contract.)

## Per-file checking instructions

There is **no external protocol/algorithm standard** governing this folder, so do not open an RFC/FIPS-compliance finding for any file here. Treat the numbered sub-claims as **OTP-internal robustness/contract checks**: for each sub-claim that could plausibly apply to the file under review, look for code that satisfies it — a `try/after` or `on_exit` that releases a captured device or monitor, a graceful branch for a corrupt manifest or bad filter, a catch-all `handle_cast`/`handle_info` clause, an enforced timeout. If the file declares such a default, guard, or cleanup, name it. If a claim clearly applies to the file (e.g. a capture module with no restoration on the exception path, or a `gen_*` callback that will `badmatch` on an unknown message) and no satisfying code is present, report it as a finding under the appropriate failure-mode class (resource leak, unhandled-message crash, error-swallowing, non-termination) — **not** as a standards-compliance violation.