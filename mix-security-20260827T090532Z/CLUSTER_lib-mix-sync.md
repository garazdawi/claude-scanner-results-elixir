# Cluster cross-file findings

#### `CSCAN-cluster-pubsub-loopback-untrusted-deserialization`

* **Finding ID** — `CSCAN-cluster-pubsub-loopback-untrusted-deserialization`
* **Severity** — Medium
* **Confidence** — medium
* **Files involved** —
  * `lib/mix/sync/pubsub.ex` — taint source + sink call: `receive_message/1` at lines 237–262 (accept at 244; read attacker length `size::unsigned-integer-64` at 246–248; read `binary` at 248; `decode_data` at 249); `decode_data/1` at 264–270 calling the sink at 266; delivery to subscriber processes at 231–234 and 161–169.
  * `lib/mix/utils.ex` (out-of-cluster helper, sink internals) — `non_executable_binary_to_term/2` at lines 29–34: `:erlang.binary_to_term(binary, opts)` with `opts` defaulting to `[]` (no `:safe`, line 30–31), and the term walker that **explicitly permits atoms** at lines 54–58 (`is_atom(other) -> other`).
  * `lib/mix/sync/lock.ex` (sibling establishing the trust posture) — `await_probe_data/1` at lines 291–304 reads only a fixed `@probe_data_size` (7-byte) payload and rejects anything that isn't the constant `@probe_data`; the moduledoc at lines 56–67 documents that a loopback peer on one of these ephemeral ports may be an *unrelated process* and must be treated as untrusted.
* **Finding shape** — Taint across files + trust-boundary/validation split (the folder’s own “loopback peer is untrusted” posture is enforced in `lock.ex` but skipped before `pubsub.ex`’s deserialization sink).
* **Evidence** —
  * `pubsub.ex:244` accepts *any* connection to the subscription port; the only “authentication” is the public constant `@tag_data` (`pubsub.ex:30`) matched at line 246. Line 246 then reads a fully attacker-controlled 64-bit `size`, line 248 does `recv(socket, size)` with no upper bound, and line 249/266 feeds the attacker bytes to `Mix.Utils.non_executable_binary_to_term(binary)`.
  * `utils.ex:30-31` shows the helper calls `:erlang.binary_to_term(binary, [])` **without `:safe`**, so atoms are interned during decode; `utils.ex:54-58` shows the post-walk accepts atoms unconditionally. The “non_executable” name therefore blocks funs/ports but *not* atom-table growth or a `:compressed` decompression bomb — neither of which is visible from `pubsub.ex`.
  * `pubsub.ex`’s own `try/rescue` in `decode_data` (264–269) does not mitigate this: atoms are interned (and a compressed term is fully expanded) *inside* `binary_to_term` before the walk, and OOM/atom-table exhaustion is not a catchable exception.
  * Contrast: `lock.ex:291-304` shows the sibling module reads a *bounded, constant-validated* 7 bytes from loopback peers precisely because (per `lock.ex:56-67`) the peer may be unrelated/hostile — the discipline `pubsub.ex` omits.
* **What the per-file pass missed** — A `pubsub.ex`-only reviewer sees a call named `non_executable_binary_to_term` and reasonably assumes it is hardened; only by reading `utils.ex` (a different file) does one learn `:safe` is not applied and atoms are permitted, and only by comparing with `lock.ex` does one see the folder itself treats these loopback peers as untrusted.
* **Suggested fix** — Treat the loopback peer as untrusted at the boundary: (1) reject `size` above a small fixed cap before `recv(socket, size)`; (2) call `Mix.Utils.non_executable_binary_to_term(binary, [:safe])` so atom interning and reference creation are blocked (the broadcast payloads are Mix’s own compile events, whose atoms already exist in the receiving node), or add a shared-secret/HMAC prefix so only same-user Mix processes are accepted, mirroring `lock.ex`’s probe contract.

#### `CSCAN-cluster-lock-ospid-terminal-injection`

* **Finding ID** — `CSCAN-cluster-lock-ospid-terminal-injection`
* **Severity** — Low
* **Confidence** — medium
* **Files involved** —
  * `lib/mix/sync/lock.ex` — `decode_lock_info/1` at lines 384–393 (`os_pid::binary` consumes the whole trailing segment of a file’s content, with no charset/length validation of the decoded value); `fetch_probe_port/1` at 256–267 and `probe/2` at 247–254 propagate that `os_pid`; `try_lock/4` at 191–193 passes it into `on_taken.(os_pid)`.
  * `lib/mix/project.ex` (out-of-cluster sink) — `on_taken` callbacks at lines 1009–1013 and 1026–1030 interpolate `#{os_pid}` directly into `Mix.shell().error([...])`, i.e. straight to the terminal.
* **Finding shape** — Taint across files (attacker-influenced file content → format/terminal-escape injection sink in a different file).
* **Evidence** — The value handed to `on_taken` is not a trusted OS PID: it is whatever bytes follow the padding in the lock file, decoded at `lock.ex:388-390` (`os_pid::binary`) from `switch_file_read/1` (446–457) of a peer-written `lock_N`/`port_P` file. `encode_lock_info/1` (364–382) caps `os_pid` to 32 bytes only on the *write* path; the *read* path applies no such bound and no filtering of control bytes. That raw binary reaches `project.ex:1012` / `project.ex:1029` via `#{os_pid}` inside `Mix.shell().error`, so ESC/BEL/CR sequences planted in a lock file are emitted verbatim to the waiting process’s terminal.
* **What the per-file pass missed** — A `lock.ex`-only reviewer sees `os_pid` handed to an opaque caller-supplied `on_taken` and cannot see it lands in a terminal `~s`/interpolation sink; a `project.ex`-only reviewer sees `#{os_pid}` and reasonably assumes (per `with_lock/3`’s doc, which calls it “the operating system PID … as a string”) that it is digits, not attacker-controlled bytes from a decoded lock file.
* **Suggested fix** — Validate `os_pid` where it is decoded (`lock.ex:384-393`): enforce the same ≤32-byte bound as the encoder and restrict it to the expected PID charset (e.g. `[0-9]+`), returning an error/`:invalidated` otherwise, so no unvalidated bytes can flow to `on_taken` and downstream terminal output.
