# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/sync/pubsub.ex
<!-- sha256: 4eb07f6c7e8f626cc962115b3ea51519124056cef40fe1ac96fdc78be0156286 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:29:54Z_

I have enough to write the report.

## Summary

`Mix.Sync.PubSub` implements cross-OS-process pub/sub over loopback TCP; the core routing is sound, but the message-receive path lacks a timeout on the payload read, so a single stalled/malformed connection permanently hangs a subscriber's entire receive loop.

## Bugfixes

#### `CSCAN-pubsub-body-recv-infinite-timeout`
- **Finding ID:** `CSCAN-pubsub-body-recv-infinite-timeout`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `pubsub.ex:248` (`receive_message/1`), via `recv/3` at `pubsub.ex:284`
- **Class:** Timeout and lifetime (infinite timeout where the caller expects to recover)
- **Impact:** After `subscribe/1` opens the shared listen socket, any local process can connect, send a well-formed 33-byte header (`"mixpubsub"` + 16-byte hash + a large `size`), and then send no body and hold the connection open. The header `recv` is guarded by a 1000 ms timeout (per the code's own comment), but the body read `recv(socket, size)` uses the default `:infinity` timeout. The `receive_message_loop` blocks forever on that one connection. Because a node uses a *single* listen socket and *one* receive loop for all its subscriptions, every subsequent pub/sub message to that node (e.g. build-lock/event notifications) is silently never delivered — a permanent, node-wide hang, with no crash to signal it. A publisher that crashes mid-`send` after the header is flushed reaches the same state without any malice.
- **Evidence:** `verified`. Reproduction mirroring `receive_message/1` (accept → 1000 ms header `recv` → `:infinity` body `recv`): the sender transmits only a valid header claiming `size=1000` and then stalls. Output:
  ```
  receiver parsed header, now blocking on body recv...
  receiver process alive after 3s of stalled body: true
  ```
  The receiver parses the header and then blocks indefinitely on the body `recv`; it is still alive and stuck 3 s later. Code confirms `recv(socket, size)` at line 248 passes no timeout, and `recv/3` (line 284) defaults `timeout \\ :infinity`.
- **Suggested fix:** Give the body read a bounded timeout, e.g. `recv(socket, size, 5_000)` (or reuse the header timeout), so a stalled/partial sender falls into the `else _other ->` branch, closes the socket, and the loop resumes accepting.

#### `CSCAN-pubsub-monitor-leak-per-subscribe`
- **Finding ID:** `CSCAN-pubsub-monitor-leak-per-subscribe`
- **Severity:** Low
- **Confidence:** high
- **Location:** `pubsub.ex:135` (`handle_call({:subscribe, ...})`)
- **Class:** Resource leak (monitors never released)
- **Impact:** `Process.monitor(pid)` is called unconditionally on every `subscribe/1`, with no dedup and no `demonitor`. A caller that subscribes to N keys (or re-subscribes to the same key) accumulates N monitors on the GenServer; when it dies, N separate `:DOWN` messages arrive, each rebuilding the whole `hash_to_pids` map. Additionally, when `ensure_socket/1` fails, the monitor established just above it is left in place even though the pid is never added to any subscription. Behaviour stays correct (the `:DOWN` handler is idempotent), so this is only wasted monitors and redundant work — not incorrect delivery — hence Low.
- **Evidence:** `asserted`. Code reading: `Process.monitor(pid)` is unconditional at line 135, occurs before the `ensure_socket` failure branch (line 155) which returns without cleanup, and no `Process.demonitor` appears anywhere in the file.
- **Suggested fix:** Monitor only when the pid is newly added to `hash_to_pids` (skip if already monitored/subscribed), and move the `Process.monitor` call after a successful `ensure_socket`.

## Risks / caveats
- The `recv`-hang finding is reachable by a misbehaving/malicious *local* process (or a publisher crashing mid-send), not by a well-formed `broadcast/2`. I treat it as in scope because the folder threat model explicitly designates `Mix.Sync.*` as the untrusted-binary surface reading bytes from other OS processes, and `subscribe/1` is the documented entry point that exposes the socket. If your scope excludes local-process adversaries entirely, downgrade it.
- I did not exercise the full GenServer end-to-end (subscribe → broadcast → deliver) under `Mix.start()`; findings rest on code reading plus the isolated socket reproduction for the hang.
- The comprehension in the `:DOWN` handler (`pids = remove_pid(...)`) initially looked like it would insert `{hash, nil}` entries; I verified by running that a falsy binding in a comprehension acts as a filter, so empty entries are correctly dropped (see Ruled out).

### Ruled out
- **`:DOWN` handler leaving `hash => nil` entries** — I suspected `pids = remove_pid(...)` returning `nil` would still emit `{hash, nil}` into the map, later crashing `MapSet.put(nil, pid)` on re-subscribe. Verified by running: a comprehension binding whose value is falsy filters the element out (`for {k,v} <- %{a: 1,b: 2,c: 3}, v2 = (if v==2, do: nil, else: v*10), into: %{}, do: {k,v2}` → `%{a: 10, c: 30}`, `b` dropped). The code is correct.
- **`String.to_integer(port)` raising in `broadcast/2`** — only reachable if the per-key dir contains a file named `port_<non-integer>`; that directory is written exclusively by this module as `port_#{port}` with an integer port, so no well-behaved path produces such a name. Not reachable via the documented API.
- **`raise File.Error` on `File.ls` error and `raise RuntimeError` on `accept` error** — these are crash-on-genuine-error paths (filesystem failure / listen socket closed), consistent with raising on invalid environment rather than a documented error-tuple contract; not a defect.
- **`recv`/`accept` `:eintr` retry dropping the original timeout to `:infinity`** — real but `:eintr` on a BEAM `gen_tcp` recv is extremely rare and the retry path is bounded in practice; folded conceptually into the body-recv timeout finding rather than reported separately.

