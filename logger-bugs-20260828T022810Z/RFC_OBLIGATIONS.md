<!-- rfc-governed: no -->
<!-- standards: -->

# Documented-contract / behavioural-obligation list — `elixir/lib/logger`

## Scope note

This folder is the **Elixir `Logger` application** (Elixir `.ex` source, not `.erl`). It is a runtime logging library layered on top of Erlang/OTP's `:logger`. It defines a primary `Logger` API, a `:logger` handler and filter (`Logger.Backends.Handler`, `Logger.Utils.translator/2`), a built-in formatter (`Logger.Formatter`), OTP translators (`Logger.Translator`), and a set of gen_server/gen_event-style backends and supervisors (`Logger.Backends.*`).

Nothing in this folder implements a network protocol, wire format, or cryptographic algorithm. Its correctness is governed entirely by **Elixir/OTP-internal contracts**: the `:logger` / `:logger_handler` / `:logger_formatter` behaviours, per-function `@spec`/`@doc` return shapes, and OTP process/supervision conventions. A per-file reviewer can derive every obligation from the code, its `@spec`s, and the OTP behaviours it plugs into — there is no external protocol/algorithm-conformance gap. Hence **rfc-governed: no**.

## Applicable standards / contracts

### External standards
None. No IETF RFC, ITU-T, W3C, ISO/IEC, NIST/FIPS, or de-facto external wire/report format governs this folder. (The mechanically-gathered standards list for the application tree was empty, and reading the sources confirms it: the format string grammar `$time $metadata[$level] $message` is an Elixir-defined convenience, not a standardized format; timestamps are Elixir/Erlang calendar terms, not an RFC 3339 wire contract this code must conform to.)

### OTP / Elixir-internal contracts (per-file context, NOT compliance obligations)
- **`:logger` handler contract** — `Logger.Backends.Handler` (and the app's handler wiring) must implement the `:logger_handler` callback shape (`log/2`, config callbacks) and must not crash the logging pipeline on a single bad event.
- **`:logger` filter contract** — `Logger.Utils.translator/2` is a `:logger` primary/handler filter: it must return `:ignore`, `:stop`, or a (possibly modified) event map, and must not raise (it already wraps translation in `try/rescue`).
- **`:logger_formatter` contract** — `Logger.Formatter.format/2` / `new/1` must return `IO.chardata()` and honor `:logger` formatter config (`format`, `metadata`, `truncate`, `colors`, timezone).
- **gen_event / gen_server backend contracts** — `Logger.Backends.Console`, `Config`, `Watcher`, `Supervisor`, `Internal` follow OTP callback shapes; `handle_info` must tolerate unknown messages; child specs / supervision restarts must be well-formed.
- **Per-function `@spec` / `@doc` return shapes** across `Logger`, `Logger.Formatter`, `Logger.Utils`.

## Checkable sub-claims

These are Elixir/OTP-internal obligations offered as guidance to the per-file reviewer. They are **not** external-standard compliance obligations and must not be adjudicated by the RFC-compliance pass.

1. **FILTER_RETURNS_LEGAL_VERDICT** — A `:logger` filter (`Logger.Utils.translator/2`) must return only `:ignore`, `:stop`, or an event map; never an arbitrary term that the logger core would mishandle. (`:logger` filter contract.)
2. **TRANSLATE_NEVER_RAISES** — Translation/formatting of a foreign or malformed log event must be caught and degraded to a `{:string, chardata}` message, not allowed to raise and drop the event or crash the handler. (See `try/rescue` in `Logger.Utils.translator/2`.)
3. **FORMATTER_RETURNS_CHARDATA** — `Logger.Formatter.format/2` and any `{module, function}` custom formatter path must return `IO.chardata()`; no bare atoms/tuples leaking to the output device. (`@spec`, `:logger_formatter` contract.)
4. **TRUNCATE_HONORED** — When a `:truncate` / message size limit is configured, formatted output must be truncated (with the documented truncation marker) rather than emitting unbounded chardata. (`Logger.Formatter` config.)
5. **METADATA_MISSING_KEY_TOLERATED** — Requested `$metadata` keys that are absent must be skipped/omitted, not cause a `KeyError`/`badmatch` in the formatter. (`Logger.Formatter` `@doc`.)
6. **HANDLE_INFO_TOLERATES_UNKNOWN** — gen_server / gen_event backends (`Logger.Backends.Console`, `Config`, `Watcher`) must not crash on unknown info messages; log-and-ignore is the contract. (`gen_server`/`gen_event`.)
7. **DOCUMENTED_RETURN_SHAPE** — Public `Logger` / `Logger.Formatter` / `Logger.Backends` functions must return the tuple/atom shape their `@spec` declares (e.g. `{:ok, formatter}` / `{:error, reason}`), with no silent demotion. (`@spec`.)
8. **DOMAIN_FILTER_STOPS_CORRECTLY** — The OTP/SASL/supervisor-report domain gating in `Logger.Utils.translator/2` must return `:stop` for the exact `:otp` / `:sasl` / `:supervisor_report` domain + config combinations and pass through others unchanged. (Filter logic.)
9. **RESOURCE_CLEANED_ON_ERROR_PATH** — Backends that open devices/ports/ETS or start watched processes must release/deconfigure them on error and on backend removal, not only on the happy path. (`Logger.Backends.Watcher` / `Supervisor`, OTP cleanup convention.)
10. **CONFIG_VALIDATION_RETURNS_ERROR** — Invalid formatter/backend configuration (bad format string, unknown option) should surface as a documented `{:error, _}` / raised `ArgumentError` at config time, not produce a silently broken handler at log time. (`Logger.Formatter.new/1` `@doc`.)
11. **COLOR_ONLY_WHEN_ENABLED** — ANSI color escapes must be emitted only when coloring is enabled/supported, so log files/pipes are not polluted with escape codes when colors are off. (`Logger.Formatter` colors option.)
12. **LEVEL_COMPARISON_RESPECTS_PRIMARY** — Events below the primary/handler configured level must be dropped before translation/formatting cost is incurred, per `:logger.get_primary_config/0` usage in `Logger.Utils`. (`:logger` level semantics.)

## Per-file checking instructions

For each sub-claim that could plausibly apply to the file under review, search for code that satisfies it. If the file declares the relevant default, config validation, `try/rescue`/`with` guard, or callback return that should satisfy the claim, name it (function + line) and treat the claim as met. If no satisfying code is present and the file is one where you'd expect the obligation enforced (e.g. a `:logger` filter that can raise, a formatter that can leak non-chardata, a gen_event backend that can crash on unknown messages, or a backend that leaks a resource on an error path), report it as a gap under the appropriate failure-mode class (crash-on-untrusted-input, resource-leak, contract-violation). Because there are **no external protocol/algorithm standards** governing this folder, do not raise RFC/standards-conformance findings here — every obligation above is an Elixir/OTP-internal contract and should be judged as such.