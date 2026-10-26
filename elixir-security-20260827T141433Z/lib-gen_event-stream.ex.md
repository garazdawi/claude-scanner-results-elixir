# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/gen_event/stream.ex
<!-- sha256: 9e3cc7dc7bfa6eb2b57f586b56a548bc8fd307a18767010c4db2dd58068bbf72 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:50:11Z_

I have enough to conclude. I read the entire 180-line file, traced its reachability (`GenEvent.stream/2` → `%GenEvent.Stream{}` → `Enumerable` protocol), and ran reproductions against a locally-built Elixir (1.21.0-dev / OTP 29) to settle the concurrency questions rather than assert them.

# Security review — `lib/elixir/lib/gen_event/stream.ex`

## Summary
This deprecated `GenEvent.Stream` module (struct + GenEvent callbacks + `Enumerable` impl) contains only message-passing/concurrency plumbing with no injection, evaluation, atom-creation, filesystem, deserialization, or crypto sinks, and its one trust-sensitive mechanism — selective `receive` keyed on an unforgeable reference — holds up under test; **no findings**.

## Security

`No findings`

The file was inventoried against every Phase 1 sink class. The only security-relevant surface is **concurrency / message passing**; every other class (code exec, compile-time exec, atom creation, command exec, file/path, archive, deserialization, template/interpolation, protocol dispatch on external type names, network, validation, crypto, unicode, shared mutable state, round-trip) has **no corresponding sink** anywhere in the 180 lines. The events flowing through `fun.(event, acc)` are pass-through data handed to the *consumer's own* reducer, not a sink.

### Ruled out

- **Event/message injection via forged mailbox messages** (`next/2`, `stream.ex:109-133`) — the selective `receive` matches `{_from, {^pid, ^ref}, …}`, binding the manager pid *and* the handler `ref`. `ref` is an Erlang reference minted by the manager in `server_add_process_handler` (`gen_event.ex:589`) and never leaves the consumer, so it is unforgeable by an outside process. Basis: **verified** — `send(self(), {:junk, {self(), make_ref()}, {:notify, :FORGED}})` placed in the consumer mailbox was *not* consumed by the stream (`take1_after_junk: [:real]`; the forged term remained as `leftover_mailbox`). Injection would require an intra-node process that already holds the real ref, which is outside the attacker model and not reachable through the public API.
- **Unbounded blocking `receive` hang** (`wait_for_handler_removal/1`, `stream.ex:159-168`, no `after` clause) — reachable via `stop/2`. The monitor set in `start/2` (`Process.monitor(pid)`) guarantees a `:DOWN` is delivered if the manager is or becomes dead, and normal handler removal delivers `:gen_event_EXIT`, so one branch always fires. Basis: **verified** — killing the manager mid-stream (`Process.exit(mgr, :kill)` while the consumer waited in `next/2` with `timeout: :infinity`) terminated the consumer with `{:killed, {Enumerable.GenEvent.Stream, :stop, …}}` rather than hanging (3 s watchdog did not trip).
- **`next/2` blocking forever on `timeout: :infinity`** (`stream.ex:134-137`) — the timeout is the value the *caller* passed to `GenEvent.stream(mgr, timeout: …)` (default `:infinity`), not attacker-derived; no trust boundary is crossed. A finite timeout deterministically `exit`s with `{:timeout, …}` as documented.
- **`flush_events/1` recursion as a CPU/DoS vector** (`stream.ex:170-178`) — bounded by `after 0`, so it drains only messages already queued for the matching `ref` at that instant and stops the moment none is immediately available; sustaining it requires possession of the unforgeable `ref` (see first item).
- **`:gen.call` to a caller-supplied `manager`** (`start/2`, `stream.ex:96-97`) — `manager` is chosen by whoever calls `GenEvent.stream/2`; pointing a stream at a hostile manager is self-inflicted, not untrusted data crossing a boundary. A malicious reply that fails the `{:ok, {pid, ref}}` match raises only in the caller's own process.
- **Always-`exit` GenEvent callbacks** (`handle_event/2`, `handle_call/2`, `stream.ex:18-35`) — the `:erlang.phash2(1,1)` switch always yields `0`, making the `:remove_handler`/`{:remove_handler, reason}` branch dead; this is a deliberate, commented Dialyzer non-local-return trick (process handlers are dispatched out-of-band and never invoke these), not a defensive gate that fails open. Not security-relevant.

## Risks / caveats
- The module is `@moduledoc false` and every function (including the `GenEvent.stream/2` entry point) is `@doc false` and deprecated, so under a strict reading there is *no* documented public API here at all; I nonetheless reviewed it as reachable via the exported, still-callable `GenEvent.stream/2` plus the `Enumerable` protocol, which is the more conservative stance.
- Reproductions ran on Elixir 1.21.0-dev / OTP 29 built from a nearby toolchain, not verified to be byte-identical to the source tree under review; the observed selective-receive and monitor semantics are OTP-level guarantees unlikely to differ across builds.
- Residual (accepted, out-of-model) risk: any co-located BEAM process that obtains the handler `ref` can inject or drain stream events. This is the standard single-node Erlang trust model, not a defect of this file.

