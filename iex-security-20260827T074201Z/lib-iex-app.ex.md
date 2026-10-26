# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/app.ex
<!-- sha256: 43d76c80663853386a4ab7d45bc16ec99beff7347b32bc7f953032499903acaf -->

_Analysed on 2026-08-27T07:47:34Z_

This file is tiny and self-contained. I've read it in full; let me work through the Phase 1 inventory against it.

The module is `IEx.App`, the OTP `Application` callback for the `:iex` application. Its only entry point is `start/2`, invoked by the OTP application controller at application boot. Both parameters (`_type`, `_args`) are ignored. Everything else in the function body is composed from compile-time constants.

Tracing each candidate sink:

- **Shared mutable state** — `Application.put_env(:stdlib, :shell_multiline_prompt, {IEx.Config, :prompt})` (line 12) writes VM-global application env. Both the key (`:stdlib`, `:shell_multiline_prompt`) and the value (`{IEx.Config, :prompt}`) are hardcoded literals. No externally derived key or value reaches it.
- **Concurrency / check-then-act** — the `with :default <- Application.get_env(...)` guard (lines 11–13) reads then writes env. It runs once, single-threaded, inside the application controller's `start/2`; the written value is constant regardless of outcome.
- **Protocol/behaviour dispatch** — `use Application` and `Supervisor.start_link/2` with a constant child list and a constant name. No dynamic module/atom.
- Code execution, compile-time execution, atom creation, command execution, file/path, archive, deserialisation, template/interpolation, network, validation, crypto, Unicode, resource consumption, round-trip — **none present**.

No external input (network, disk, dependency, archive, git ref, build input) crosses into this file through any path. There are no sinks to drive, so there is nothing behavioural to reproduce.

## Summary
`IEx.App` is a minimal OTP `Application` callback that starts the IEx supervision tree and installs a constant multiline-prompt config value; it contains no externally influenced sink.

## Security

No findings

### Ruled out
- **`Application.put_env(:stdlib, :shell_multiline_prompt, {IEx.Config, :prompt})` (line 12) — shared mutable state.** Writes VM-global app env, but both key and value are hardcoded literals; no externally derived data reaches the key or value, so no cross-tenant poisoning is possible. The cross-application write to `:stdlib` is the documented integration point between IEx and the OTP shell.
- **`with :default <- Application.get_env(...)` guard (lines 11–13) — check-then-act / dead defensive code.** This is a "don't clobber an operator-configured prompt" guard, not a security gate; it runs exactly once during single-threaded application start, and the value written is constant either way, so no race and no bypass has security consequence.
- **`start/2` arguments (`_type`, `_args`).** Supplied by the OTP application controller, not an attacker, and ignored here regardless.

## Risks / caveats
The security-relevant behaviour of IEx (arbitrary code evaluation, pry consent, broker takeover, `.iex.exs` auto-evaluation, remote-shell trust) lives in `evaluator.ex`, `server.ex`, `broker.ex`, `pry.ex`, and `helpers.ex` — not here. This file only wires up the supervision tree (`IEx.Config`, `IEx.Broker`, `IEx.Pry`) and a static config default; reviewing it in isolation cannot surface those cross-file concerns, and none of them are introduced by anything in `app.ex`.

