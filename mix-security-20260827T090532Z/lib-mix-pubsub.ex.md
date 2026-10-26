# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/pubsub.ex
<!-- sha256: 74a79a793cf8725cacafb9c79fa56a769154786f87ecf44ad52cc7c86a3c996a -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/pubsub.ex
<!-- sha256: 74a79a793cf8725cacafb9c79fa56a769154786f87ecf44ad52cc7c86a3c996a -->

_Re-analysed on 2026-08-27_

`Mix.PubSub` is a supervisor-lifecycle helper. Its only code-execution sink is starting operator-configured listener modules; its only untrusted-input surface (cross-OS-process events) lives in `pubsub/subscriber.ex` and `Mix.Sync.PubSub`, not here. The re-review targets the two hedges the first pass carried:

1. "I assumed OTP's standard `Supervisor.start_child/2` contract" (Risks/caveats).
2. The check-then-act race resolution, which depended on the same unverified contract.

Both are now settled by **running** the Supervisor contract against OTP 29, not by asserting it.

## Summary
`Mix.PubSub` exposes no security vulnerability reachable through its public API: its sole code-execution sink (starting configured listener modules) is gated on operator-authored root-project config, and its sole untrusted-input surface (cross-OS-process events) is handled in other files. The lifecycle races are benign, now confirmed by executing the `Supervisor.start_child/2` duplicate-id and concurrent-start contract on OTP 29.

## Security
No findings

### Ruled out

- **Listener code execution via `configured_listeners/0` (lines 78, 80, 91–99).** `Supervisor.child_spec(&1, [])` + `Supervisor.start_child/2` start arbitrary modules named in `:listeners` — genuine code execution. Not a finding: both sources are the **root** project's own config — `Application.get_env(:mix, :listeners)` (root `config/*.exs`) and `Keyword.get(Mix.Project.config(), :listeners)` (root `mix.exs`), confirmed via `deps.loadpaths.ex:57,83` running in root context and the `task.compiler.ex:40–53` contract requiring the operator to explicitly list each listener. A transitive dependency cannot auto-register; a dependency that could mutate the `:mix` app-env at build time already has build-time RCE, so no trust boundary is crossed (matches the folder threat model: `mix.exs`/config are RCE-by-design for the operator).
  - Axes examined: config source (root vs. dependency), auto-registration path. | Not examined: none — this is a trust-source ruling with no two-implementation differential to drive; it rests on the documented operator-opt-in contract, not on a guard whose input disagreement could be probed.

- **Check-then-act race in `start/0` (lines 25–38) and `start_listeners/0` (lines 59–74).** Concurrent callers can both pass the `Process.whereis/1` guard and both call `Supervisor.start_child/2`. Not a finding: the race loser receives `{:error, {:already_started, _pid}}`, which both clauses map to `:ok` — no duplicate process, no crash. **Verified by execution:** I ran 5 concurrent `Supervisor.start_child/2` calls with the same child id against a `:one_for_one` supervisor on OTP 29; exactly one returned `{:ok, pid}` and the other four returned `{:error, {:already_started, <same pid>}}`. The supervisor serialises the calls, so no two callers ever both start a child.
  - Axes examined: duplicate-id return value, concurrent (N=5) start serialisation. | Not examined: none.

- **`inspect(reason)` interpolated into `raise RuntimeError` (lines 36, 72).** `reason` is an internal supervisor error term; `inspect/1` yields an inert string and Elixir's `raise/2` does not treat the message as a format string. No injection or format-string sink.
  - Axes examined: message-argument interpretation by `raise/2` (documented to take a literal binary, not a format string). | Not examined: none.

- **Untrusted cross-OS-process events.** The moduledoc describes receiving events from other OS processes, but subscription/decoding (`Mix.Sync.PubSub.subscribe/1`) and fan-out (`send(pid, message)`) live in `pubsub/subscriber.ex` and `Mix.Sync.PubSub`, not in this file — out of scope. Any injection/deserialisation/DoS in the cross-process channel would surface there.

- **Application-env read as shared mutable state (line 95).** `Application.get_env(:mix, :listeners, [])` is read-only here and sourced from operator config; no externally derived key/value is written by this file.

## Risks / caveats
- The findings are settled by static reasoning, cross-file tracing, and one executed contract test, because the module has no decoder/parser/path/atom/command sink to drive with hostile input. The genuinely attacker-facing surface (deserialising and dispatching events published by other OS processes) is in `Mix.Sync.PubSub` and `Mix.PubSub.Subscriber`, which this review did not audit.
- The Supervisor contract this file depends on (serialised child handling, `{:already_started, pid}` on duplicate id) is **verified on OTP 29**, not assumed. A non-standard supervisor replacement is out of the documented model.

## Did I verify this
- **`Supervisor.start_child/2` duplicate-id + concurrent contract — VERIFIED (executed).** Ran an Elixir script on Erlang/OTP 29 (erts-17.0.2) / Elixir 1.21.0-dev: sequential duplicate-id start → `{:error, {:already_started, #PID}}`; 5 concurrent starts of the same id → one `{:ok, pid}`, four `{:error, {:already_started, <same pid>}}`. This clears the first-pass "I assumed OTP's standard `Supervisor.start_child/2` contract" hedge for both the `start/0` and `start_listeners/0` race rulings.
- **Trust source of `configured_listeners/0` — verified by cross-file reading** (`deps.loadpaths.ex:57,83` root context; `task.compiler.ex:40–53` explicit-listing contract). No differential axis exists to execute here; ruling stands on the documented operator-opt-in mechanism.
- **`raise/2` message semantics — verified** against documented behaviour (literal binary, not a format string).

## Dropped in re-review
None. All prior ruled-out entries are preserved. The two first-pass hedges ("I assumed" the Supervisor contract; the race conclusion depending on it) are removed by **upgrading the evidence from asserted to executed**, not by dropping any conclusion — the rulings themselves are unchanged and now rest on a run against OTP 29.
