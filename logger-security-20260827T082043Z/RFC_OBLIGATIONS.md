This is Elixir's Logger library — application-level logging infrastructure. No network protocol, wire format, or cryptographic algorithm is implemented here.

<!-- rfc-governed: no -->
<!-- standards: -->

# Compliance obligation list — `elixir/lib/logger`

## Applicable standards / contracts

### External standards
None. This folder implements Elixir's `Logger` — an application logging facility (public API, backend/handler plumbing, message formatting, and OTP-report translation). It does not implement any IETF RFC, ITU-T/W3C/ISO-IEC document format, or NIST/FIPS cryptographic algorithm. It does not speak syslog-on-the-wire (RFC 5424/3164), emit a structured interchange format like GELF/CEF, or perform any negotiation or cryptographic operation. The formatter produces free-form human-readable text and Elixir-internal term structures, not a standardized report schema consumed by third-party tooling. Therefore the compliance pass has nothing to adjudicate.

### OTP-internal contracts (per-file context, NOT compliance obligations)
These are guidance for the per-file reviewer only — do **not** treat them as external standard-conformance obligations:

- **`Logger` public API contracts** (`logger.ex`) — level ordering (`debug < info < notice < warning < error < critical < alert < emergency`, mapped onto `:logger` levels), and macro/function `-spec` return shapes.
- **`:logger`/`error_logger` behaviour integration** (`backends/handler.ex`, `translator.ex`, `app.ex`) — the OTP `logger` handler callback shape and `report_cb`/translator conventions.
- **`gen_event` backend behaviour** (`backends/console.ex`, `backends/config.ex`, `backends/watcher.ex`) — the documented `Logger` backend callbacks (`init/1`, `handle_event/2`, `handle_call/2`, `code_change`).
- **Formatter contract** (`formatter.ex`, `utils.ex`) — the documented `format/4` / compile-pattern shapes and metadata handling; correctness here is an OTP/API contract, not a wire standard.

## Checkable sub-claims

Because the folder is not governed by any external protocol/algorithm standard, there are **no external RFC/NIST/CIS conformance sub-claims** to enumerate. Any defects in this folder (crashes on malformed metadata, unbounded formatting, level-comparison mistakes, message truncation, injection of untrusted data into log output, ReDoS in format-pattern parsing, etc.) are ordinary correctness / secure-coding issues and are already covered by the per-file security and correctness review — they must not be re-labeled as RFC-compliance gaps.

## Per-file checking instructions

Treat this folder as **out of scope for the compliance pass**: there are no external standard obligations to match against. The per-file reviewer should ignore the (empty) external-standards list and instead rely on the normal security/correctness review, using the OTP-internal contracts above only as background context (e.g. to recognize the expected backend/handler/formatter callback shapes). Do not raise "RFC/NIST compliance" findings for this folder; report any genuine problems under their ordinary sink/failure-mode classes instead.