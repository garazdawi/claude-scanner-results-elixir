# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/broker.ex
<!-- sha256: 9b31bdd30244a2360749e49d13df05b71f3790099f7e5d95ed76a78d7326d213 -->

_Analysed on 2026-08-27T07:50:02Z_

## Summary
`IEx.Broker` is an internal session-takeover coordinator whose only externally-influenced input is the fully-trusted distribution cluster (via `erpc`/`Node.list`), and it exposes no code-execution, atom-creation, path, deserialization, or interpolation sink to untrusted data through a documented API — no in-scope security finding.

## Security

No findings.

### Ruled out

- **`servers/1` erpc fan-out to `Node.list()` (lines 167–174).** The only value entering this file from outside the local node is the result of `:erpc.call(node, IEx.Broker, :shell, [])` across the cluster. The module and function are hard-coded literals (no dynamic dispatch), and the trust boundary crossed is distributed-Erlang cluster membership, which the folder threat model treats as fully trusted (a co-node sharing the cookie can already halt/inspect any node). A malicious node could return a non-pid to make `Process.monitor(pid)` raise, or make the call hang/raise (`:erpc.call/4` defaults to `:infinity` and raises `{:erpc, :noconnection}` on disconnect), crashing/blocking the local `Broker` `handle_call({:take_over,...})` — but that is squarely the trusted-cluster boundary and thus out of scope. Not reachable with network/disk/dependency data through a documented public API.
- **`take_over/3` `location`/`whereami`/`opts` (lines 81–116).** These originate from `IEx.Pry` / `Macro.Env` (local source-file trust) and are only *forwarded* verbatim in a `send/2` message. No interpolation, IO, atom creation, or dispatch on them occurs in this file (that happens in `server.ex`), so there is no sink here.
- **`for ... pid = :erpc.call(...)` comprehension (lines 169–171).** Looks like a filter that could misbehave, but it is the intended assign-as-filter idiom: nodes whose `shell/0` returns `nil` are dropped (verified: `for x <- [:a,:b,:c], y=(if x==:b, do: nil, else: x), do: y` ⇒ `[:a, :c]`). `shell/0` only ever returns a pid or `nil`, so no valid value is wrongly kept/dropped. Correct as written.
- **`{:accept,...}` takeover acceptance / double-accept (lines 124–133).** Accepting requires knowing the takeover `ref`, which is the unguessable `reference()` component of the take_over caller's GenServer `from` tag, transmitted only to the discovered servers/shells. `pop_in` makes acceptance single-shot (second accept ⇒ `{:error, :already_accepted}`). No auth bypass and no double-accept race.
- **`register/1` accepting an arbitrary pid (lines 55–58, 118–122).** A local co-tenant process could register a bogus pid to receive take_over broadcasts, but `IEx.Broker` is `@moduledoc false` internal coordination, this is local-process (not network/disk/dependency) input, and any co-tenant BEAM process already has full VM access — not a meaningful boundary under the review scope.
- **`{:DOWN,...}` handling / use-after-death (lines 143–165).** `handle_info` pops the dead server from `servers` and `refuse/4`s it out of every pending takeover, replying `{:error, :refused}` when a takeover's last server dies. No stale-pid reply and no double-reply path survives.
- **`respond/4` boolean guard (lines 67–73).** Matching on literal `true`/`false` can only be defeated by a caller violating the `boolean()` spec — an uninteresting contract violation, not a finding.

## Risks / caveats

- **`shell/0` and `evaluator/1` `MatchError` on a dead-pid race (lines 28, 39).** `Process.info(pid, :dictionary)` returns `nil` (not `{:dictionary, _}`) if the target dies between the lookup (`:user_drv.whereis_group()` / arg) and the `Process.info` call, so `{:dictionary, dictionary} = ...` raises `MatchError` (verified: `Process.info(dead_pid, :dictionary)` ⇒ `nil`, match ⇒ `%MatchError{term: nil}`). Since `shell/0` is invoked remotely via `erpc` in `servers/1`, a race on any cluster node can propagate a crash into the local `Broker`'s take_over handler and restart it (pending takeovers lost, sessions survive). This is a robustness/availability blemish gated behind the trusted-cluster boundary, not an in-scope security defect; a defensive `case Process.info(pid, :dictionary) do {:dictionary, d} -> d[:shell]; _ -> nil end` would harden it.
- **Monitor accumulation for shells in `servers/1` (line 171).** Registered servers reuse their `register`-time monitor, but shells are `Process.monitor`ed afresh on every `take_over` and are never demonitored on `:accept`/`:refuse` (only implicitly cleared when the shell actually dies). Repeated pry/takeover activity across a cluster slowly accumulates monitors on the Broker and shell pids. The trigger is developer-initiated (pry), not attacker-controlled, and growth is bounded by live pids — a minor resource leak, noted for completeness rather than as a finding.

