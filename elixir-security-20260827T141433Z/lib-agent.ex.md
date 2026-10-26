# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/agent.ex
<!-- sha256: 89540a0778f286a6f5d8be6e4fe675c7db709b2d24692959e28985be71e2a346 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T08:19:45Z_

I've read the entire file. It's the `Agent` standard-library module — a thin, documented wrapper over `GenServer`. Let me record my analysis.

## Summary

`Agent` is a thin pass-through wrapper around `GenServer` (`start*`, `get*`, `update*`, `cast`, `stop`) with no parsing, no byte→atom/term/path conversion, and no dangerous sink reachable through its documented public API; no meaningful security findings.

## Security

`No findings`

Every public function forwards caller-supplied terms to `GenServer` without transforming external bytes into a dangerous form. I traced each candidate sink class from the Phase 1 inventory against the full file:

| Class | Present in file? | Trace |
|---|---|---|
| Code execution | Only by design | `get/5`, `get_and_update/5`, `update/5`, `cast/4`, `start*/4` forward `{module, fun, args}` inside a `GenServer` message. The actual `apply` runs in `Agent.Server` (not under review). `module`/`fun`/`args` are supplied by the caller as the *documented contract*; there is no external-bytes→callable conversion in this file. |
| Atom creation | No | No `String.to_atom`, `List.to_atom`, `:erlang.binary_to_atom`, or `Module.concat` anywhere. |
| Compile-time execution | No | `__using__/1` only assembles a `child_spec/1` map from developer-written `use Agent, opts`; `Macro.escape(opts)` escapes the value, and the body touches no filesystem/network. |
| Deserialisation / command / file / path / archive / template / Base / URI / crypto / regex | No | None of `binary_to_term`, `System.shell`/`os.cmd`, `File.*`/`Path.*`, `:erl_tar`/`:zip`, `EEx`, `Base.decode*`, `URI.parse`, `:crypto`, or `Regex.compile` appear in the file. |
| Shared mutable state / concurrency | Only by design | The agent *is* shared mutable state; its race semantics are documented and caller-managed. No externally-derived key/value poisons process/VM-global state from within this file. |

### Ruled out

- **MFA-executing functions as a code-execution sink (`get/5`, `get_and_update/5`, `update/5`, `cast/4`, `start_link/4`, `start/4`)** — Basis: file read in full; these send `{op, {module, fun, args}}` as a `GenServer` message with the MFA taken *directly from the caller's arguments*. Executing the given MFA is the documented purpose (mirrors `apply/3`); no untrusted-byte decoding happens here, and a caller forwarding attacker-controlled module/fun is a caller-side injection, explicitly out of scope. The `apply` itself lives in `Agent.Server`, a different file.
- **Atom-table exhaustion** — Basis: full read; no string/binary→atom conversion of any kind exists in the file.
- **Compile-time / build-machine execution via `use Agent`** — Basis: `__using__/1` body inspected; it builds a static child-spec map and calls `Supervisor.child_spec/2` with `Macro.escape`d opts, performing no I/O. Input is the developer's own `use` line, not a dependency-controlled value.
- **Unquote hygiene in `__using__/1`** — Basis: the only `unquote` is `unquote(Macro.escape(opts))`, which injects an escaped literal value (not caller AST); no expression is spliced in.
- **Missing `is_function` guards on `cast/4` and `start*/4`** — Basis: these take MFA, not a closure, so an arity guard is inapplicable; not a security defect.

## Risks / caveats

- **Threat-model / standards mismatch.** The standing folder threat model and the entire Unicode/RFC obligation checklist (UAX #31/#39/#55, bidi, NFC, escape validation) target the compiler front-end in `elixir/lib/elixir/src` (tokenizer/interpolation). This file is `elixir/lib/elixir/lib/agent.ex`, a stdlib module in a different directory; none of those obligations apply to it, so none were checkable here. That is a scoping observation, not a gap.
- **Cross-file boundary.** The security-relevant behavior of the MFA path (the `apply` of `{module, fun, args}` against agent state, and any handling of anonymous-function version skew noted in the "distributed agents" docs) executes in `Agent.Server`, which was not in scope. I did not review it; if that module trusted the message shape improperly it would be a finding there, not here.
- **Nothing to reproduce.** Because no decoding/parsing sink exists in the file, there was no hostile input to drive; the "No findings" conclusion rests on a complete read of all 518 lines (the bulk of which is documentation) rather than on a behavioral experiment. Confidence is high given the file's small, wrapper-only surface.

