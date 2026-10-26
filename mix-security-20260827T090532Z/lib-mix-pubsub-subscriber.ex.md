# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/pubsub/subscriber.ex
<!-- sha256: c33214801763e9b984637ff8fa80d1000dd09274c530e372784a36f3ebf17111 -->

_Analysed on 2026-08-27T09:52:49Z_

Verified. The accumulation is unbounded (100k messages → 100000-element list, no cap) and double-flush raises `Protocol.UndefinedError` (`Enum.reverse(nil)`). I have enough to write the report.

## Summary

`Mix.PubSub.Subscriber` is a thin relay GenServer that buffers pub/sub events and forwards them to listener processes; it contains no code-exec, atom, path, or command sinks, and its one security-relevant property is an unbounded pre-flush buffer fed by an externally injectable message stream.

## Security

#### `CSCAN-subscriber-unbounded-preflush-buffer`
- **Finding ID** — `CSCAN-subscriber-unbounded-preflush-buffer`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `elixir/lib/mix/lib/mix/pubsub/subscriber.ex:41-44` (`handle_info/2`, the pre-flush accumulate clause; sink at line 43 `update_in(state.acc, &[message | &1])`)
- **Class** — Resource consumption
- **Impact** — Between `Mix.PubSub.start/0` and `Mix.PubSub.start_listeners/0`, the subscriber buffers every received event in `state.acc` with no size or count cap. That window deliberately spans a full dependency compilation (`pubsub.ex` comment: "a compilation may be required before the listener modules are available… The subscriber is going to accumulate events and reply them once the listeners are started"), i.e. potentially minutes. The events originate from `Mix.Sync.PubSub`, which delivers them over a loopback TCP socket (`sync/pubsub.ex:9,26-28,231-235`); the socket is not access-controlled, the target hash is the directory name `Base.url_encode64(md5(build_path))` and the ephemeral port is the `port_<port>` filename under a world-listable `System.tmp_dir!()` subdir. A co-resident local process can therefore flood the port with events (each up to an attacker-chosen size via the `size::unsigned-integer-64` framing) during the compile window, growing the `acc` list without bound until the Mix build process exhausts memory. Amplifies a stream of small local-network messages into unbounded resident memory (build DoS). Local-attacker-only, self-machine.
- **Evidence** — `verified` (in-file behavior): driving the real callback with 100 000 messages showed the buffer growing 1:1 with no cap —
  ```
  $ elixir /tmp/t.exs
  after 2 msgs (accumulating): %{acc: [{:big, "xxxxxxxxxx"}, :evt1]}
  acc length after 100k injected msgs: 100000
  ```
  The injection vector (loopback TCP reachability / hash+port discoverability) is reasoned from `sync/pubsub.ex`, not driven end-to-end (`asserted`).
- **Suggested fix** — Bound the buffer: cap `acc` at a fixed length (or total byte budget) in the accumulate clause and drop-oldest (or drop-newest) once exceeded, so a pre-listener flood cannot grow memory without limit. Buffering to await listeners is fine; the missing cap is the defect.

### Ruled out
- **Double-flush crash (`Enum.reverse(nil)`).** `handle_cast(:flush, %{acc: nil})` evaluates `Enum.reverse(nil)` and raises `Protocol.UndefinedError` (verified). Not a security finding: it is triggered only by calling `flush/0` a second time (a plain cast), which `Mix.PubSub.start_listeners/0` guards against with `Process.whereis(...)`, and which no attacker-controlled *message* can reach (events arrive via `handle_info`, never `handle_cast`). Internal-API-misuse / latent robustness bug only.
- **`IO.warn(message, [])` (line 30).** `message` is the fixed `{:error, "failed to subscribe to Mix events using TCP, reason: #{inspect(reason)}"}` string from `Mix.Sync.PubSub.subscribe/1`; `IO.warn` is not a format function and `inspect` neutralizes the reason. No injection.
- **`send(pid, message)` after `Supervisor.which_children` (lines 53-59).** Children come from the trusted `Mix.PubSub.ListenerSupervisor`; `is_pid/1` filters `:restarting`/`:undefined`; the which_children→send TOCTOU is benign (send to a dead pid returns `:ok`). No injectable path component.
- **Relaying attacker-influenced `message` to listeners.** This file forwards the term verbatim; any dangerous interpretation lives in the listener modules (`IEx.MixListener`, project-configured `:listeners`), which are out of this file's scope.

## Risks / caveats
- I did not run the full end-to-end loopback injection (craft `<<"mixpubsub", md5(build_path)::16, size::64, term>>` to the subscriber's port); the sink's unbounded growth is verified directly against the real callback, but reachability from an external loopback peer is reasoned from `Mix.Sync.PubSub`.
- The actual message-injection primitive and the per-message size bound (`size::unsigned-integer-64` + `Mix.Utils.non_executable_binary_to_term/1`) live in `elixir/lib/mix/lib/mix/sync/pubsub.ex`, which is outside this review; whether that transport should be access-controlled is a finding about that file, not this one.
- Safety of what listeners do with attacker-influenceable event content was not assessed here (different files); this file only relays.

