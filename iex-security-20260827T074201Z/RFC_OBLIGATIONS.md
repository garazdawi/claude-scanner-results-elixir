This is IEx, Elixir's interactive shell — application/tooling code with no wire protocol or cryptographic algorithm.

<!-- rfc-governed: no -->
<!-- standards: -->

# Compliance scope for `elixir/lib/iex`

This folder implements **IEx**, Elixir's interactive REPL/shell: the read-eval-print loop, expression evaluator, autocompletion, shell helpers, evaluation history, the `pry`/breakpoint debugger, and the shell server/broker that manages interactive sessions. None of it implements a standardized network protocol, wire format, or cryptographic algorithm. Its correctness is governed by Elixir/OTP-internal contracts and general secure-coding practice — both of which the per-file security review already covers directly — not by any external RFC / ITU-T / W3C / ISO / NIST / FIPS obligation.

## Applicable standards / contracts

### External standards
None. No IETF RFC, ITU-T recommendation, W3C recommendation, ISO/IEC standard, NIST SP / FIPS publication, or de-facto external wire/report format constrains this folder. The compliance pass has nothing to adjudicate here.

### OTP / Elixir-internal contracts (per-file context, NOT compliance obligations)
These are guidance for the per-file reviewer only — they are internal conventions, not external-standard obligations, and the compliance pass must not treat them as such:

- **GenServer / behaviour callbacks** — `IEx.Server`, `IEx.Broker`, `IEx.Config`, and `IEx.MixListener` are OTP servers; their `init/1`, `handle_call/3`, `handle_cast/2`, `handle_info/2` return-shape contracts are OTP-internal.
- **Application callback** — `IEx.App` implements the `Application` behaviour (`start/2`).
- **`@spec` / `@doc` return-shapes** — public helper functions in `IEx.Helpers`, `IEx.Introspection`, `IEx.Info`, and `IEx.Autocomplete` document return shapes as internal API contracts.
- **Evaluation-context/state conventions** — `IEx.Evaluator` and `IEx.History` maintain binding/history state per OTP conventions, not a wire format.

## Checkable sub-claims

None derived from external protocol/algorithm standards. There are no MUST / MUST NOT / SHOULD NOT wire-format, negotiation, state-machine, or algorithm-correctness clauses for a per-file reviewer to check against this folder. Any concerns (e.g. code injection through evaluated input, unsafe atom creation in autocompletion, arbitrary-code execution via `pry`) are ordinary secure-coding / CWE-class issues already handled by the per-file security review, not RFC/NIST-conformance items.

## Per-file checking instructions

Treat this folder as **not RFC-governed**: the downstream compliance pass should perform no standard-conformance adjudication here, because the external-standards list is empty. The per-file security reviewer should proceed with its normal secure-coding review (input handling in the evaluator, atom/table exhaustion in autocomplete and history, process/message-handling races in the server/broker, and the trust boundary around `IEx.Pry` breakpoints) and should NOT report "missing RFC/NIST obligation" findings against any file in this folder — the OTP-internal contracts listed above are context only, not compliance gaps.