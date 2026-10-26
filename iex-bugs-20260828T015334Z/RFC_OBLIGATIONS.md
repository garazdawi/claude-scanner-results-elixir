Confirmed: IEx is Elixir's interactive shell — GenServer-based runtime/utility code with no external protocol or algorithm standard.

<!-- rfc-governed: no -->
<!-- standards: -->

# Documented-Contract / Behavioural-Obligation List — `elixir/lib/iex`

## What this folder is

This is **IEx**, Elixir's interactive shell (`Interactive Elixir`). The modules are runtime/utility code: a shell server (`IEx.Server`), an evaluator loop (`IEx.Evaluator`), a takeover broker (`IEx.Broker`, a `GenServer`), tab-completion (`IEx.Autocomplete`), expression history (`IEx.History`), the `pry`/breakpoint machinery (`IEx.Pry`), config (`IEx.Config`), help/introspection (`IEx.Introspection`, `IEx.Helpers`, `IEx.Info`), and the OTP application (`IEx.App`). There is **no** on-the-wire protocol, no wire/file format consumed by third-party tools, and no cryptographic algorithm. There are **no** standards-data tables in or beside the folder (only Elixir source, `mix.exs`, and ExUnit tests).

## Standards references accounted for

The scan reported **"(no standards references found in the application tree)"**, and reading a representative subset (`server`, `broker`, `autocomplete`, `evaluator`, `iex`) confirms it: the code implements the Elixir/OTP shell, not any IETF/ITU/W3C/ISO/NIST/FIPS specification. There is therefore **no external protocol/algorithm standard** to enumerate, and the RFC-governance verdict is `no`. The only "standards" that touch this code are OTP-internal conventions (`gen_server`/`gen_statem` behaviour callbacks, `-spec`/typespec/`@doc` contracts) and generic secure-coding practice — both explicitly excluded from the compliance pass by the instructions above.

## Applicable standards / contracts

### External standards
None. The `<!-- standards: -->` marker is intentionally empty. Nothing in this folder implements a network protocol, wire/document format, or standardized algorithm whose normative clauses a single-file reviewer could not derive from the code itself.

### OTP-internal contracts (per-file context, NOT compliance obligations)
These are guidance for the per-file reviewer only — **not** standard-conformance obligations:

- **`GenServer` behaviour** — `IEx.Broker` (and any other `use GenServer` module) must implement `handle_call/3`, `handle_cast/2`, `handle_info/2`, `init/1` return shapes correctly and tolerate unexpected messages without crashing.
- **Typespec / `@spec` contracts** — e.g. `IEx.Server.run_opts`, `IEx.Broker.take_ref`/`shell`. Public functions should return the shapes their specs declare.
- **`@doc`/`@moduledoc` documented behaviour** — e.g. `IEx.Evaluator` "trapping exceptions in the code being evaluated", `IEx.Server` group-leader I/O responsibilities.
- **Application/supervision contract** — `IEx.App` as an OTP application callback.

## Checkable sub-claims

These are OTP-internal / secure-coding checks the per-file reviewer may apply; **none** is an external-standard compliance obligation.

1. **HANDLE_INFO_TOLERATES_UNKNOWN** — `GenServer` callbacks (`IEx.Broker`) must not crash on unknown `handle_info`/`handle_call` messages; ignore or log. (`gen_server(3)`.)
2. **DOCUMENTED_RETURN_SHAPE** — Functions must return the tuple/type their `@spec` declares; no extra elements or silent demotion. (Typespec.)
3. **EVAL_TRAPS_EXCEPTIONS** — The evaluator must trap exceptions raised by user-entered code and continue the loop, not let them kill the shell session. (`IEx.Evaluator` `@doc`.)
4. **RESOURCE_CLEANED_ON_ERROR_PATH** — Monitors/refs/processes taken during `pry`/takeover (`IEx.Broker`, `IEx.Pry`) must be demonitored/released on every error and normal-exit path. (`try/after` / OTP convention.)
5. **MONITOR_REF_CLEARED** — `take_ref`/`reference()` handles created for takeover must be cleared when the taking process dies (`DOWN` handling), preventing stale-ref leaks. (OTP monitor convention.)
6. **GROUP_LEADER_IO_ONLY** — Shell I/O must go through the group leader as documented, not directly to `:stdio`, so remote/nested shells behave correctly. (`IEx.Server` `@doc`.)
7. **NO_RAISE_ON_MALFORMED_INPUT** — Autocomplete/introspection entry points fed malformed or partial input should return an empty/`{:no, …}` result rather than raise. (`IEx.Autocomplete`, `IEx.Introspection` `@doc`.)
8. **HISTORY_BOUNDED** — Expression history must honour its configured size limit and not grow unbounded. (`IEx.History` / `IEx.Config`.)
9. **CONFIG_DEFAULTS_VALIDATED** — `IEx.Config` must supply defaults and reject/normalise invalid option values rather than propagate them into the eval loop. (`IEx.Config` contract.)
10. **PRY_RESTORES_STATE** — After a `pry`/breakpoint session ends (normally or by error), the evaluator must restore normal evaluation and not leave the server wedged. (`IEx.Pry`/`IEx.Server` `@doc`.)

## Per-file checking instructions

Treat this folder as **out of scope for the external-standards compliance pass** — the `<!-- standards: -->` marker is empty and there is no protocol/algorithm to adjudicate. For per-file review, use the OTP-internal sub-claims above only as context: for each sub-claim that could plausibly apply to the file in hand, look for code that satisfies it (a `try/after` releasing a monitor, a catch-all `handle_info`, a spec-matching return, input validation before use). If the file declares a default, guard, or cleanup that satisfies the claim, name it. If a claim is clearly relevant (e.g. a `GenServer` with no catch-all clause, or a takeover path that leaks a monitor on error) and no satisfying code is present, report it as an ordinary correctness / resource-leak / crash-safety finding under the appropriate failure-mode class — **not** as an RFC/standards-conformance violation.