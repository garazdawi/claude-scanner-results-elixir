# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/pubsub/subscriber.ex
<!-- sha256: c33214801763e9b984637ff8fa80d1000dd09274c530e372784a36f3ebf17111 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/pubsub/subscriber.ex
<!-- sha256: c33214801763e9b984637ff8fa80d1000dd09274c530e372784a36f3ebf17111 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`Mix.PubSub.Subscriber` is a small internal GenServer that buffers pub/sub events until listeners start, then forwards each buffered event (and every subsequent event) to the listener children. On re-review I read the two neighbour modules the first pass had only assumed about — `Mix.PubSub` (`pubsub.ex`, the sole caller of `flush/0`/`start_link/1`) and `Mix.Sync.PubSub` (`sync/pubsub.ex`, the source of the messages that land in `handle_info`) — and executed the ordering/`nil` claims. I still find **no reachable behavioural bug**. The prior report's two open hedges are both resolved to verified; no new findings.

## Bugfixes

No findings

## Risks / caveats

- The module is `@moduledoc false`; its only intra-Mix "public API" is `start_link/1` and `flush/0`, both driven exclusively by `Mix.PubSub.start`/`start_listeners` (`pubsub.ex:28`, `:57`–`:75`). Re-read confirms `flush/0` is called at exactly one site (`pubsub.ex:64`), only in the `{:ok, _pid}` branch of `Supervisor.start_child`, i.e. once per process lifetime (see Ruled out).
- **(Resolved — was hedged in first pass)** The message-delivery contract of `Mix.Sync.PubSub.subscribe/1` is now read, not assumed. `subscribe/1` (`sync/pubsub.ex:46`) registers `self()` (the Subscriber) in `hash_to_pids`; the socket receive loop forwards `{:message, hash, message}` to `Mix.Sync.PubSub` (`sync/pubsub.ex:233`), whose `handle_info/2` does `send(pid, message)` (`sync/pubsub.ex:161`–`166`) — delivering the **bare decoded term** (`Mix.Utils.non_executable_binary_to_term/1`, `sync/pubsub.ex:266`) to the Subscriber's mailbox. That term can be *any* shape, which is exactly why the Subscriber's `handle_info/2` is a total catch-all (`subscriber.ex:36`, `:41`). The Subscriber monitors nothing and is not the monitor owner (the monitor at `sync/pubsub.ex:135` is held by `Mix.Sync.PubSub`, so `{:DOWN, …}` never reaches this module), so no unmodelled system message can arrive. Forward-everything design confirmed against the actual sender.

### Ruled out

- **Second `flush/0` crashes the subscriber (`Enum.reverse(nil)`)** — After the first `flush`, `state.acc` is `nil` (`subscriber.ex:49`); a second `:flush` cast would evaluate `Enum.reverse(nil)`, raising `Protocol.UndefinedError`. Executed evidence: `Enum.reverse(nil)` → `Protocol.UndefinedError` (re-run this pass). **Not reachable**: `Mix.PubSub.start_listeners/0` calls `Subscriber.flush()` only inside the `{:ok, _pid}` branch of `Supervisor.start_child` (`pubsub.ex:62`–`65`); every later `start_listeners/0` short-circuits on `Process.whereis(Mix.PubSub.ListenerSupervisor)` (`pubsub.ex:59`) or hits `{:already_started, _}` (`pubsub.ex:67`) and returns `:ok` without flushing. So `flush/0` fires at most once. Latent fragility (a defensive `acc: nil` clause on `handle_cast` would harden it) but no in-scope trigger.
  - Axes examined: reachability of a second flush via all `start_listeners/0` return branches; the `Enum.reverse(nil)` failure mode itself (executed). | Not examined: none.
- **`handle_cast`/`handle_info` non-total (CaseClause/FunctionClause)** — `handle_info/2` has a catch-all second clause (`subscriber.ex:41`) covering every non-`nil`-`acc` state; state is always `%{acc: list | nil}`; the only sender (`Mix.Sync.PubSub`, `sync/pubsub.ex:161`) delivers an arbitrary bare term, all of which both clauses accept. `handle_cast/2` only ever receives `:flush` — the only cast in the tree (`grep` of the folder shows `flush/0` at `subscriber.ex:18` as the sole `GenServer.cast` to this name). No unhandled shape reachable.
  - Axes examined: message shape (arbitrary decoded term, traced to sender); state shape (`list`/`nil`); cast tag (`:flush` only, whole-tree). | Not examined: none.
- **`Supervisor.which_children` on a not-yet-started `ListenerSupervisor`** — `notify_listeners/1` (`subscriber.ex:52`) is reached either from `handle_cast(:flush, …)` (runs only after `start_listeners/0` started `ListenerSupervisor` and *then* cast `:flush`, `pubsub.ex:62`–`64`) or from the `acc: nil` `handle_info` clause (`acc` becomes `nil` only via that same flush, `subscriber.ex:49`). In both paths `Mix.PubSub.ListenerSupervisor` is registered before the call, so `which_children/1` is never invoked against a missing name.
  - Axes examined: both entry paths into `notify_listeners/1` and the ordering of supervisor-start vs. flush-cast at the caller. | Not examined: none.
- **Event ordering / duplication across the buffer→live switch** — Buffered events are prepended (`[message | &1]`, `subscriber.ex:43`) then `Enum.reverse`-d on flush (`subscriber.ex:48`), giving chronological delivery; executed: `reduce([:a,:b,:c],[],&[&1|&2]) |> Enum.reverse` → `[:a, :b, :c]` this pass. Post-flush events (`acc: nil`) are delivered one at a time (`subscriber.ex:37`). Each listener receives each event once. The prepend/reverse axis is now executed and correct.
  - Axes examined: prepend-then-reverse chronological ordering (executed). | Not examined: concurrent mailbox interleaving of the `:flush` cast against in-flight `handle_info` deliveries under a genuine multi-OS-process publisher — this relies on BEAM single-mailbox FIFO and cannot be exercised deterministically in this sandbox, so it stays untried. Entry remains ruled out on the strength of the FIFO guarantee, but this axis is not executed.
- **`flush/0` cast when subscriber failed to start (`init` → `:ignore`)** — On `Mix.Sync.PubSub.subscribe/1` returning `{:error, _}`, `init/1` warns and returns `:ignore` (`subscriber.ex:29`–`31`), so `@name` is unregistered. `GenServer.cast/2` to an unregistered name is fire-and-forget, returns `:ok` without raising, and no events would have been delivered anyway. Not a crash, not a behavioural defect.
  - Axes examined: `init/1` failure branch; `GenServer.cast` to an unregistered name. | Not examined: none.

## Did I verify this

- **Read** `subscriber.ex` (full), `pubsub.ex` (full — sole caller of `flush/0`/`start_link/1`), and `sync/pubsub.ex` (full — the actual message sender feeding `handle_info`). The first pass had only assumed the last two; both are now read.
- **Executed** in the sandbox's Elixir toolchain: `Enum.reverse(nil)` → `Protocol.UndefinedError`; `reduce([:a,:b,:c],[],&[&1|&2]) |> Enum.reverse` → `[:a,:b,:c]`.
- **Not executed**: the concurrent buffer→live interleaving under a real multi-OS-process publisher (recorded as the single remaining `Not examined` axis above; rests on BEAM mailbox FIFO, not on execution).

### Dropped in re-review

- **Caveat: "did not exhaustively audit `Mix.Sync.PubSub.subscribe/1`'s message-delivery contract."** Dropped as an open hedge and **upgraded to verified** in Risks/caveats above. New evidence: reading `sync/pubsub.ex:46`, `:134`–`166`, `:231`–`270` shows the Subscriber is registered as a subscriber pid and receives the bare decoded term via `send(pid, message)`; it is not the monitor owner, so no `{:DOWN, …}` or other unmodelled message reaches it. This matches the forward-everything design of `notify_listeners/1`, and the Subscriber's catch-all `handle_info/2` accepts any such term — no unhandled-shape defect exists in either module. No finding is warranted, so nothing is upgraded to a low-confidence finding.
