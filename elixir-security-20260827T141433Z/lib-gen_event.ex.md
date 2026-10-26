# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/gen_event.ex
<!-- sha256: 47c760d2f137d5d99b000acb3ec68d59aa54e7f0dc0cc75e4a61bbc581d8e413 -->
<!-- tree: 3389f8405 2026-08-25 -->

# Security review — `gen_event.ex`
<!-- sha256: 47c760d2f137d5d99b000acb3ec68d59aa54e7f0dc0cc75e4a61bbc581d8e413 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28. This pass reads the callgraph neighbour `lib/elixir/lib/gen_event/stream.ex` to eliminate the "outside this file" hedge from the first pass._

## Summary
`GenEvent` is a deprecated Erlang-style event-manager behaviour whose exported functions perform only message passing and by-design behaviour dispatch. No sink (code-exec, atom creation, path/command/deserialisation, crypto) is reachable with attacker-controlled data through its documented public API. The one first-pass residual — a possible unbounded wait in the sync/ack streaming path — is now **verified bounded** by reading the neighbour module, and is downgraded to a fully-reasoned ruled-out entry (no longer a hedged residual).

## Security

**No findings.**

Every dangerous-looking sink dispatches on modules/targets the *caller* supplies as part of the documented contract (a handler module, or a `{:via, module, term}` / `{atom, node}` manager reference), or is reachable only by hand-crafting the manager's undocumented internal message protocol — both explicitly out of scope.

### Ruled out

- **`apply(mod, fun, args)` in `do_handler/3` (line 843) — arbitrary MFA dispatch.** `mod` is the handler module registered via `add_handler/3`, `add_mon_handler/3`, or `swap_handler/5`; `fun` is a fixed callback name (`:init`/`:handle_event`/`:handle_call`/`:handle_info`/`:terminate`). Invoking `AnyModule.init/1` requires calling `GenEvent.add_handler/3` with a chosen module — i.e. you already execute Elixir in the same VM. Intended behaviour-dispatch model (identical to `:gen_event`/`GenServer`), not a defended boundary. *Basis: reasoned from call graph.* **Confidence: high.**

- **`mod.send(name, …)` in `notify/2` `{:via, mod, name}` clause (line 236) — dynamic module dispatch.** `mod` is the registration-module component of the caller-supplied `{:via, module, term}` manager reference (per the `name`/`manager` type at lines 82/86). Not attacker-controlled unless the caller forwards an untrusted manager ref, which is a caller contract violation. *Basis: reasoned from the exported type contract.* **Confidence: high.**

- **Manager process crash via malformed internal-protocol message.** A message shaped `{_from, tag, {:sync_notify, event}}` (line 388) with a non-`{from, ref}` `tag` reaches `reply/2` (line 775), which pattern-matches `{from, ref}` and raises `FunctionClauseError`, killing the manager. **This reproduces**, but requires crafting a raw internal message; the documented public path (`sync_notify/2` → `rpc/2` → `:gen.call/4`, line 315) always builds a well-formed `{from, ref}` tag, so the public path is safe. Crafting internal-process messages is out of scope. *Basis: verified by execution in the first pass —*
  ```
  manager alive after public sync_notify? true             # public API: safe
  manager alive after crafted bad-tag sync_notify? false   # crafted internal msg only
  ```
  **Confidence: high (reproduced).**

- **`{:DOWN, ref, …}` / `{:add_process_handler, …}` handling (lines 393, 418) — handler removal / process-handler injection by spoofed messages.** Manager internal protocol, not reachable through any documented `GenEvent.*` function; monitor refs are unforgeable. Out of scope. *Basis: reasoned.* **Confidence: high.**

- **`:infinity` timeout in `rpc/2` (line 315).** `add_handler`, `sync_notify`, `ack_notify`, `remove_handler`, `swap_handler`, `which_handlers` block forever. `:gen.call/4` monitors the target and returns on its death, so a dead manager cannot wedge the caller; a slow/hung handler blocking the manager is the documented no-back-pressure property, and the caller (not an untrusted party) chose `:infinity`. *Basis: reasoned; standard OTP pattern.* **Confidence: high.**

- **Sync/ack streaming collection — untimed `receive` in `server_collect_process_handlers/5` (lines 689–706).** *(Upgraded from hedged assumption in first pass — the first pass wrote "I did not drive the full streaming path (it lives outside this file)"; I have now read the neighbour producer.)* For `:sync`/`:ack` modes the manager blocks in an untimed `receive` awaiting `{^ref, :ok}` from each process handler. I read `gen_event/stream.ex`, the sole creator of process handlers, and traced the ref plumbing end-to-end:
  - The manager creates the handler with `ref = Process.monitor(pid)` where `pid` is the stream consumer (`server_add_process_handler/3`, lines 588–592), and that same `ref` is the value awaited in the collect loop (line 691).
  - The consumer (`GenEvent.Stream` `Enumerable.reduce` → `next/2`, stream.ex:106–138) receives `{_from, {^pid, ^ref}, {…notify, event}}` and, via `wrap_reducer/1` (stream.ex:76–92), sends `{ref, :ok}` back — for `:ack` before invoking the user fun (line 79), for `:sync` in an `after` block that runs even if the user fun raises (lines 85–90).
  - Because the manager **monitors the consumer** (`ref` *is* the monitor ref), a dead/exited consumer produces `{:DOWN, ^ref, …}` (line 702), which bounds the wait; and a raising user reducer still fires the `after send(manager, {ref, :ok})`.

  The only way the collect loop blocks indefinitely is a consumer whose *user reducer function loops forever without raising* — application-supplied, trusted code, matching the documented no-back-pressure semantics of Erlang `:gen_event`. The party who can trigger the collect (`sync_notify`/`ack_notify`) is itself blocked in `rpc/2` and is not the party that causes the hang. No attacker-controlled input reaches an unbounded wait. *Basis: verified by reading the neighbour producer `gen_event/stream.ex` and tracing the monitor-ref plumbing across both files.* **Confidence: high.**

- **`report_error/5`, `report_status/2`, `format_status/2` log handler state and the full process dictionary via `:error_logger` (lines 890, 901, 527).** State/pdict are the application's own trusted data; redaction is available through the handler's `format_status/2` hook. Not attacker-influenced. *Basis: reasoned.* **Confidence: high.**

- **Dead `{:remove_handler, {:bad_call, msg}}` branch in `__using__` default `handle_call` (line 127).** `:erlang.phash2(1, 1)` is always `0`, so the branch never runs — an intentional Dialyzer workaround (comment at line 121), not a security defence that fails to fire. *Basis: reasoned.* **Confidence: high.**

- **`notify/2` swallows all errors and returns `:ok` (lines 226–241).** Matches the declared fire-and-forget contract `@spec notify(manager, term) :: :ok` (line 222). Not a security defect. *Basis: reasoned.* **Confidence: high.**

- **Atom creation, `Code.eval*`, `System.cmd`/shell, `File`/path ops, `binary_to_term`, `Base`/`URI` round-trips, crypto comparisons.** None present. *Basis: full read of the 909-line file.* **Confidence: high.**

## Risks / caveats

- **Trust model.** Findings were judged against the exported `GenEvent.*` functions as the public API (all `@doc false`/`@deprecated`, but callable). Per the ground rules, the ability to send arbitrary Erlang messages to the manager PID, or to call `add_handler` with an attacker-chosen module, is treated as already-inside-the-VM access, not a defended boundary.

- **Unicode / RFC compliance checklist is not applicable to this file.** The standing folder threat model and the UAX #31 / UTS #39 / UAX #9 (Trojan-Source) obligations target the compiler front-end in `elixir/lib/elixir/src` (tokenizer, interpolation). `gen_event.ex` performs no source-text, identifier, normalization, or bidi handling, so none of those clauses govern it — not an unchecked gap.

- **Verification scope.** The internal-protocol manager-crash reproduction was run in the first pass against the built Elixir 1.21.0-dev / OTP 29 toolchain. The streaming-collection bound was verified this pass by reading `gen_event/stream.ex` and tracing the monitor-ref plumbing across both files (not separately executed — the bound is a structural monitor guarantee, established by the `Process.monitor` at line 589 matching the `{:DOWN, ^ref, …}` clause at line 702, not an input-dependent behaviour). Remaining ruled-out items are reasoned from the call graph and exported type contracts.

## Did I verify this?
- **Streaming untimed-`receive` bound** — **Verified** by reading the callgraph neighbour `gen_event/stream.ex` and confirming the manager monitors the consumer (`ref = Process.monitor(pid)`, line 589) so consumer death yields a bounding `{:DOWN, ^ref, …}` (line 702), and that the consumer's `:sync` acknowledgement fires from an `after` block (stream.ex:88–89) even when the user fun raises. No neighbour-behaviour assumption remains.
- **Manager crash on malformed internal tag** — **Verified** by execution (first pass).
- **All other ruled-out entries** — reasoned from the in-file call graph and the module's exported type/`@spec` contracts; no cross-file behavioural assumption is relied upon.

## Dropped in re-review
Nothing dropped. The first-pass "Streaming selective-receive not exercised end-to-end" residual is **not dropped** — it is retained and upgraded to a verified ruled-out entry (see the sync/ack streaming bullet above), because reading `gen_event/stream.ex` confirmed the wait is monitor-bounded and the only unbounded case is trusted application reducer code, consistent with documented no-back-pressure semantics.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Dead `{:remove_handler, {:bad_call, msg}}` branch in the `__using__` default `handle_call` (line 127).** `:erlang.phash2(1, 1)` is always `0`, so the branch never runs — but this is an intentional Dialyzer workaround documented by the preceding comment (line 121), not a security defence that fails to fire. *Basis: reasoned.*
