# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/sync/pubsub.ex
<!-- sha256: 4eb07f6c7e8f626cc962115b3ea51519124056cef40fe1ac96fdc78be0156286 -->

_Analysed on 2026-08-27T10:16:55Z_

All behaviors are verified against the built-from-source OTP 29 / Elixir 1.21.0-dev toolchain. Here is the report.

## Summary

`Mix.Sync.PubSub`'s receive path decodes bytes from an unauthenticated loopback TCP socket with an unsafe `binary_to_term` and unbounded length/timeout, giving any local process several ways to crash or freeze the Mix BEAM node.

## Security

#### `CSCAN-pubsub-unsafe-binary-to-term`
- **Finding ID** — `CSCAN-pubsub-unsafe-binary-to-term`
- **Severity** — High
- **Confidence** — high
- **Location** — `elixir/lib/mix/lib/mix/sync/pubsub.ex:266`, `decode_data/1` (via `Mix.Utils.non_executable_binary_to_term/1`)
- **Class** — Deserialisation / Atom creation (CWE-502, CWE-770)
- **Impact** — The subscriber's receive loop calls `non_executable_binary_to_term(binary)` with the default `opts = []`, so `:erlang.binary_to_term/2` runs **without `:safe`** on bytes taken straight off the socket. Any process that can reach the loopback listen port (discoverable via the world-readable `port_<N>` file — dir `0755`, file `0644` in shared `/tmp` — or by scanning) can send `"mixpubsub"` + 16 arbitrary bytes + size + a term encoding many novel atoms. Atoms are interned during decoding, *before* the `non_executable_terms/1` walk runs (and that walk explicitly *accepts* atoms, so it provides no protection). The atom table is global and never GC'd; ~1,048,576 atoms aborts the entire VM ("no more index entries in atom_tab"), unrecoverable. No key/hash secret is needed — decoding happens regardless of the routing hash. This crashes any BEAM running Mix (a `mix test`, a language server, a dev server) at a co-located or cross-user attacker's will.
- **Evidence** — `verified`. `Mix.Utils.non_executable_binary_to_term(<<131,119,byte_size(name)::8>> <> name)` created **1** new atom (`atoms created: 1`); a single 5,088,902-byte payload created **200,000** atoms (`created: 200000`, `atom table limit: 1048576`), so ~5 such messages (or one ~26 MB message; `size` is a 64-bit field) exhausts the table. The same novel-atom bytes under `[:safe]` in a fresh VM were rejected: `{:rescued, ArgumentError, "...invalid or unsafe external representation of a term..."}`.
- **Suggested fix** — Decode with `:safe`: `Mix.Utils.non_executable_binary_to_term(binary, [:safe])`. This blocks new-atom creation (and external fun/port refs) at decode time.

#### `CSCAN-pubsub-decompression-bomb`
- **Finding ID** — `CSCAN-pubsub-decompression-bomb`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `elixir/lib/mix/lib/mix/sync/pubsub.ex:246-249` (`size::unsigned-integer-64` → `recv(socket, size)` → `decode_data/1`)
- **Class** — Resource consumption (decompression bomb)
- **Impact** — The external term format supports compressed terms, and `binary_to_term` inflates them with no size ceiling; `:safe` does **not** bound decompression. The on-wire `size` is a full 64-bit value with no cap, and no cap is applied to the decoded term. A small compressed payload expands ~1000× (the format's 4-byte uncompressed-size field allows up to ~4 GB per term), so an unauthenticated local peer can drive multi-GB allocation and OOM-kill the Mix node, before the `non_executable_terms` walk ever runs.
- **Evidence** — `verified`. A 64 MiB binary compressed to 65,246 wire bytes; `Mix.Utils.non_executable_binary_to_term(comp)` returned a 67,108,864-byte term (`amplification: 1028.6x  (mem +64 MiB)`). With `[:safe]` it still inflated to 67,108,864 bytes (`:safe does NOT bound decompression`). `:gen_tcp.recv(sock, 2^64-1, _)` is accepted (returns `{:error, :timeout}` only when data is withheld), so the size field is unbounded in practice.
- **Suggested fix** — Enforce a small maximum on `size` immediately after parsing the header (drop the connection if exceeded), and reject compressed external terms (external-format bytes beginning `131, 80`) / bound the decoded term size before use.

#### `CSCAN-pubsub-infinite-body-recv`
- **Finding ID** — `CSCAN-pubsub-infinite-body-recv`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `elixir/lib/mix/lib/mix/sync/pubsub.ex:248` (body `recv(socket, size)` → `recv/3` default `:infinity`, line 284), driven by the serial `receive_message/1` loop (231-262)
- **Class** — Resource consumption / Concurrency (unbounded blocking recv, no per-connection isolation)
- **Impact** — The header recv has a 1 s timeout (line 247), but the body recv uses the default `:infinity` timeout, and `receive_message/1` handles one connection at a time with no per-connection spawn. A single client that sends a valid 33-byte header claiming an N-byte body and then withholds the body (or dribbles it slowloris-style) blocks the loop forever. Because this is the sole receive loop for *all* of that node's subscriptions, message delivery for every subscriber stops indefinitely while one attacker holds a connection open — a zero-cost, unauthenticated liveness DoS on Mix's cross-process eventing.
- **Evidence** — `verified`. In a faithful replica of the loop (1 s header timeout, `:infinity` body timeout, serial accept), an attacker socket sent `"mixpubsub" + <<0::128>> + <<100::64>>` and 0 body bytes; a subsequent well-formed 5-byte publisher was never served: output `BLOCKED: legit publisher never served while attacker stalls mid-body`.
- **Suggested fix** — Give the body recv a finite timeout (as the header already has) and/or an overall per-connection deadline, and handle each accepted socket in its own spawned process so one stalled peer cannot block the accept loop.

#### `CSCAN-pubsub-broadcast-port-parse-crash`
- **Finding ID** — `CSCAN-pubsub-broadcast-port-parse-crash`
- **Severity** — Low
- **Confidence** — high
- **Location** — `elixir/lib/mix/lib/mix/sync/pubsub.ex:71-72`, `broadcast/2` (`port = String.to_integer(port)`)
- **Class** — Unclassified (untrusted-filename → integer-parse crash)
- **Impact** — `broadcast/2` lists the subscription directory and does `String.to_integer/1` on the suffix of every `port_*` entry with no validation. A file named `port_evil` (or `port_` + a huge/blank string) makes `broadcast/2` raise `ArgumentError`, aborting whatever build step is publishing (it is called from `Mix.Tasks.Deps.Compile` and `Mix.Task.Compiler`, i.e., during `mix compile` / `deps.compile`). The directory path is predictable — `System.tmp_dir!()` resolves to shared `/tmp`, and `mix_pubsub_user<uid>` uses the victim's real uid — so a local attacker who pre-creates (squats) the guessed `<hash>` directory world-writable can plant a poison `port_*` file and crash the victim's build; the non-`port_` names are safely filtered, only the bad `port_` suffix trips the parser.
- **Evidence** — `verified`. Running `broadcast/2`'s generator over `["port_123","somethingelse","port_evil","README"]` filtered the non-matches but raised on `port_evil`: `{:raised, ArgumentError, "...not a textual representation of an integer..."}`. `System.tmp_dir!()` → `/tmp`, dir mode `755`, file mode `644` (world-readable/traversable, and `/tmp` world-writable).
- **Suggested fix** — Parse with `Integer.parse/1` and skip entries that don't yield a valid port in `1..65535`, e.g. filter the generator on a successful parse rather than calling `String.to_integer/1`.

### Ruled out
- **Huge `size` pre-allocation** — `:gen_tcp.recv/3` with a ~1 TB or 2⁶⁴−1 length does **not** pre-allocate; it returned `{:error, :timeout}` with negligible memory delta (164 KB). The memory-DoS route is the decompression bomb, not the length argument. (The stall angle of the same `recv` is covered by `CSCAN-pubsub-infinite-body-recv`.)
- **Path traversal on receive** — the 16-byte `hash` from the socket is used only as a `hash_to_pids` map key, never as a filesystem path; `path/1` derives directories from `Base.url_encode64(md5, padding: false)` (22-char url-safe alphabet, no `/` or `..`). No traversal.
- **`broadcast` message tampering to attacker's socket** — redirecting a victim's `broadcast` to an attacker-controlled `port_<N>` leaks only `term_to_binary` of trusted local Mix events and requires the same `/tmp`-squat as the parse-crash finding; low marginal value, folded into that finding's context.

## Risks / caveats
- **Message spoofing/injection into listeners** — a decoded term is forwarded via `send(pid, message)` to listener processes when the 16-byte hash matches `hash_to_pids` (hash = `md5(build_path)`). An attacker who knows/guesses the victim's `build_path` can inject arbitrary non-executable terms (fake compiler events, unexpected messages) into listener mailboxes. I did not build an end-to-end listener to characterize downstream impact; the atom/decompression/stall DoS findings need no such hash and are the higher-confidence issues.
- **Cross-user reach depends on `tmp_dir`** — port-file discovery via `/tmp` assumes a shared temp dir and default umask (verified here: `/tmp`, `0755`/`0644`). Under `PrivateTmp` or a per-user `TMPDIR` the file is not cross-user readable, but the loopback listen port is still reachable by any local process via a port scan + the public `"mixpubsub"` tag, so the DoS findings do not depend on the file's permissions.
- I verified atom-creation *rate* and the 1,048,576 limit rather than actually aborting the test VM (which would kill the shell); the terminal crash is the documented consequence of exceeding `atom_limit`.

