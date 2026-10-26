<!-- rfc-governed: no -->
<!-- standards: -->

# ExUnit (Elixir unit-testing framework) — standards applicability

## Applicable standards / contracts

### External standards
None. This folder implements **ExUnit**, Elixir's built-in unit-testing framework (test case registration, assertions, callbacks, doctests, output formatting, IO/log capture, a failures manifest, and a CLI formatter). It does not implement any network protocol, wire format, or cryptographic algorithm governed by an IETF RFC, ITU-T/W3C/ISO-IEC recommendation, or NIST/FIPS publication.

Notably, ExUnit's built-in formatters (`lib/ex_unit/cli_formatter.ex`, `lib/ex_unit/formatter.ex`) emit **human-readable console output only** — there is no JUnit/Surefire XML reporter in this tree (that is provided by the separate third-party `junit_formatter` package). So no de-facto external report-schema obligation applies either. The only `TestCase`/`testcase` occurrences are Elixir struct names, not XML elements.

### OTP-internal contracts (per-file context, NOT compliance obligations)
These are guidance for the per-file reviewer only; the compliance pass must not adjudicate them:
- **Formatter behaviour** — custom formatters are `GenServer`/`GenEvent`-style handlers receiving the event stream (`:suite_started`, `:test_finished`, `:case_finished`, etc.) from `event_manager.ex`; return-shape and message-tag contracts are ExUnit-internal.
- **Assertion return contract** — `ExUnit.Assertions` macros must raise `ExUnit.AssertionError` on failure and return the truthy value otherwise; this is a documented ExUnit `@doc`/`-spec`-style contract, not an external standard.
- **Callback ordering** — `setup`/`setup_all`/`on_exit` ordering and supervision (`callbacks.ex`, `on_exit_handler.ex`) follow ExUnit's documented semantics.
- **Failures manifest format** — `failures_manifest.ex` serializes a term to disk for `--failed`; it is a private on-disk format consumed only by ExUnit itself, not an interoperable external schema.

## Checkable sub-claims
None derived from external protocol/algorithm standards. There are no RFC/NIST/FIPS MUST/MUST-NOT/SHOULD-NOT wire-behaviour or algorithm-correctness clauses that constrain this folder. Any defects here (crashes, incorrect assertion diffing, capture races, manifest corruption) are ordinary correctness/robustness issues already covered by the per-file review, not standards-conformance gaps.

## Per-file checking instructions
Treat this folder as **application/utility code with no external-standard compliance obligations**. The compliance pass should not open findings against any RFC/NIST/FIPS/ITU-T/W3C clause for these files, because none govern a unit-testing framework. Per-file reviewers should proceed with the normal correctness, robustness, and secure-coding review (bad-input handling, races in capture/server processes, manifest read/write integrity, resource cleanup in callbacks) and use the OTP-internal contracts above only as background context — not as standard-conformance checks.