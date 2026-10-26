# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/broker.ex
<!-- sha256: 9b31bdd30244a2360749e49d13df05b71f3790099f7e5d95ed76a78d7326d213 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/broker.ex
<!-- sha256: 9b31bdd30244a2360749e49d13df05b71f3790099f7e5d95ed76a78d7326d213 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:02:00Z; re-reviewed 2026-08-28 (hedge-resolution pass)_

## Summary
`IEx.Broker.servers/1` (reached by the documented `take_over/3`) both leaks a monitor per shell on every takeover and can crash the entire broker when any connected node fails the remote `:erpc.call`. Both findings are now backed by executed evidence rather than reasoning.

## Bugfixes

#### `CSCAN-broker-erpc-remote-crash`
- **Finding ID** — `CSCAN-broker-erpc-remote-crash`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `broker.ex:167-174`, `servers/1` (invoked from `handle_call({:take_over, …})`, `broker.ex:102`)
- **Class** — Error-handling asymmetry (unguarded remote call)
- **Impact** — When the node is clustered with any peer that does not answer `:erpc.call(node, IEx.Broker, :shell, [])` successfully — a plain Erlang node (no `Elixir.IEx.Broker` module → `:undef`), a node without IEx started, or a node that disconnects between `Node.list()` and the call (`:noconnection`) — `:erpc.call` re-raises the failure. The `pid = :erpc.call(...)` comprehension binding does **not** trap the raise (a raised exception in a comprehension filter propagates; the binding only filters falsy *return values*). `servers/1` has no surrounding `try`, so the exception propagates out of the `{:take_over, …}` `handle_call`, crashing the singleton `IEx.Broker` GenServer. The supervisor restarts it with empty `servers`/`takeovers`, so every previously-registered IEx server loses its takeover registration, and the caller's `take_over/3` (a `:infinity` call) exits instead of getting a `{:error, _}` tuple. Every subsequent `pry`/takeover that scans that peer crashes the broker again.
- **Suggested fix** — Wrap the remote lookup so a failing node is skipped rather than fatal, e.g. `pid = (try do :erpc.call(node, IEx.Broker, :shell, []) rescue _ -> nil catch _, _ -> nil end)`, keeping the falsy-filter behaviour of the comprehension binding.
- **Threat note** — Distributed Erlang is an explicit trust boundary (peers are only as trusted as the cookie). A connected non-IEx or flaky peer node — attacker-influenced or merely a mixed-language cluster — turns any local `pry`/takeover into a broker crash for all shells on the node. Reached through the `@doc`/`@spec`'d public `take_over/3`.
- **Evidence** — `verified` (executed on OTP 29 / Elixir 1.21):
  - `:erpc.call(node(), NoSuchModuleXYZ, :shell, [])` → raises `ErlangError` (the `:undef` peer case).
  - `:erpc.call(:"nonode@nohost-xyz", IEx.Broker, :shell, [])` → raises `ErlangError` with message `Erlang error: {:erpc, :noconnection}` (the disconnect-mid-scan case).
  - `for x <- [1,2,3], pid = (raise "boom"), do: {pid, x}` → the raise **propagates** out of the comprehension (result `:propagated`), confirming the filter binding does not shield `servers/1` from a failing remote call.
  - `servers/1` has no surrounding `try` (read of full module).

#### `CSCAN-broker-shell-monitor-leak`
- **Finding ID** — `CSCAN-broker-shell-monitor-leak`
- **Severity** — Low
- **Confidence** — high
- **Location** — `broker.ex:167-174`, `servers/1` (monitor created at line 171)
- **Class** — Resource leak (monitor never released)
- **Impact** — Every `take_over/3` calls `servers/1`, which does `Process.monitor(pid)` for each live shell pid (local `user_drv` group plus every remote node's shell). These monitor refs are used only as transient `server_ref`s inside `state.takeovers[ref]`; they are never stored in `state.servers` and never `Process.demonitor`ed on the accept (`broker.ex:124-133`), refuse (`broker.ex:135-165`), or DOWN (`broker.ex:143-153`) paths — accept discards them via `pop_in`, refuse discards via `List.delete`/`pop_in`, and DOWN pops `state.servers[server_ref]` (where shell refs are never stored). So each takeover permanently leaks one monitor per reachable shell onto the (long-lived) shell process and inside the broker. In a session with many `pry`/takeover cycles these accumulate without bound. When such a shell eventually dies it also delivers one redundant `:DOWN` per leaked monitor, all handled harmlessly but wastefully.
- **Suggested fix** — Demonitor the shell refs once a takeover resolves — e.g. flush them with `Process.demonitor(server_ref, [:flush])` for every `server_ref` in the popped takeover on the accept path (and drop the remaining ones on refuse/DOWN) — or track shell monitors separately from registered-server monitors so they can be cleaned up.
- **Threat note** — Not attacker-reachable in a meaningful sense; a local correctness/resource bug driven by ordinary `pry`/takeover usage via the public `take_over/3`.
- **Evidence** — `verified` (executed simulation of the `servers/1` monitor lifecycle): a long-lived "shell" process monitored five times with the refs discarded (as accept/refuse/DOWN do) leaves `Process.info(self(), :monitors)` reporting **5** live monitors toward that shell; the leaked monitors persist even after a *separate* later batch is `Process.demonitor(_, [:flush])`ed (count stays at the 5 leaked ones). This directly demonstrates that discarding a monitor ref without `Process.demonitor` accumulates monitors. Corroborated by static fact: the module contains no `Process.demonitor` call and never stores the `servers/1`-created shell refs beyond the transient `takeovers` entry, which `pop_in`/`List.delete` discard without demonitoring. (A full end-to-end repro against a real `:user_drv` shell was not stood up headless, but the leak mechanism itself is now executed, not merely reasoned.)

### Ruled out

- **`servers/1` crashing on `Process.monitor(nil)` when `shell()` returns `nil`** — the comprehension binding `pid = :erpc.call(...)` acts as a filter for falsy values, so `nil` is dropped before `Process.monitor`.
  - Evidence: `verified` — `for x <- [1,2,3], y = (if rem(x,2)==0, do: x*10, else: nil), do: y` → `[20]`, the `nil` filtered out, no crash (executed).
  - Axes examined: falsy (`nil`) return value of the filter binding. | Not examined: none (the only way this crashes is a `nil` reaching `Process.monitor`, and the falsy-filter axis is the sole path to that; tried and confirmed dropped).

- **`evaluator/1` `MatchError` when `Process.info(pid, :dictionary)` returns `nil`** — `nil` is only returned for a dead pid; `shell()` returns a live group pid or `nil`, and `evaluator/1` guards `nil` with `if pid`. Passing a dead pid violates the caller contract.
  - Evidence: `verified` for the value axes — `Process.info(live_pid, :dictionary)` → `{:dictionary, [...]}` (matches), `Process.info(empty_dict_pid, :dictionary)` → `{:dictionary, []}` (matches), `Process.info(dead_pid, :dictionary)` → `nil` (would `MatchError`) — all executed.
  - Axes examined: live pid with populated dictionary; live pid with empty dictionary; already-dead pid. | Not examined: the TOCTOU race where `pid` is live when passed/guarded but dies before `Process.info` runs (a concurrent-death interleaving I did not execute). Because that axis is untried, this entry stays `asserted`. Note the same unguarded `{:dictionary, dictionary} = Process.info(pid, :dictionary)` shape exists in `shell/0` (`broker.ex:28`) between `:user_drv.whereis_group()` and the `Process.info` call; the race there is likewise untried.

- **Reply-once / takeover bookkeeping on accept vs refuse vs DOWN** — accept (`broker.ex:124-133`) pops the takeover once and replies once; refuse (`broker.ex:135-141`) and DOWN (`broker.ex:143-153`) funnel through `refuse/4` (`broker.ex:155-165`), which replies to `from` exactly when `server_refs` drains to `[]` and otherwise re-stores the pruned list. No double-reply or missed-reply path found by tracing the three sites.
  - Evidence: `asserted` — source-trace only; the GenServer was not driven with a fake multi-server registration to exercise the interleavings.
  - Axes examined (by reading): single-server accept; single-server refuse; single-server DOWN. | Not examined: concurrent multi-server interleavings — e.g. a `:DOWN` for one `server_ref` arriving after a partial `refuse` has already pruned the list, or an `:accept` racing a `:DOWN` on the same `ref`. These interleavings were not executed against the real GenServer, so this entry stays `asserted`.

## Risks / caveats
- The whole module is `@moduledoc false`; findings are scoped to its `@doc`/`@spec`'d functions (`take_over/3`, `respond/4`, `register/1`, `shell/0`, `evaluator/1`), treated as the module's public surface per the review instructions.
- Both bugfix findings are now backed by executed evidence: the `:erpc` crash by direct `:erpc.call` invocations (`:undef` and `:noconnection`) plus a comprehension-propagation test, and the monitor leak by an executed simulation of the `servers/1` monitor lifecycle. Neither depended on reading a neighbour module to reach `verified`.
- The `:erpc` crash requires cluster topology: it is triggered by a *connected peer* that fails the call. On a single node only the local `node()` is scanned, and `IEx.Broker` is always loaded there, so the failure surfaces only once a failing peer is connected — a scoping fact about when it fires, not a reduction in confidence that it fires.

### Did I verify this?
- `:erpc.call` failure modes (`:undef`, `:noconnection`) and their propagation — **yes, executed** (OTP 29 / Elixir 1.21).
- Comprehension filter behaviour for both falsy returns (dropped) and raised exceptions (propagated) — **yes, executed**.
- `Process.info(_, :dictionary)` return shapes for live/empty/dead pids — **yes, executed**.
- Monitor accumulation without `Process.demonitor` — **yes, executed** via a faithful lifecycle simulation; full `:user_drv`-backed end-to-end repro — **no** (impractical headless), but the leak mechanism itself is executed.
- Reply-once GenServer state machine under concurrent multi-server interleavings — **no**, source-trace only; that axis remains untried and the entry stays `asserted`.
- The `shell/0`/`evaluator/1` TOCTOU race (pid dies between liveness and `Process.info`) — **no**, untried.

### Dropped in re-review
None. Every prior finding and ruled-out entry is retained; the two bugfix findings were strengthened from mixed/`asserted` evidence to `verified` by execution, and each ruled-out entry gained an explicit `Axes examined … | Not examined …` field. No dismissal was moved from a weaker to a stronger state without executed evidence for the axis in question.
