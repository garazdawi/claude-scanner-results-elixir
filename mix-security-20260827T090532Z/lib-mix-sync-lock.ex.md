# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/sync/lock.ex
<!-- sha256: 24bd8881a53da76d828820c5377f175cfc8fe1d997de2ce39e29c82406bc25cb -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/sync/lock.ex
<!-- sha256: 24bd8881a53da76d828820c5377f175cfc8fe1d997de2ce39e29c82406bc25cb -->

_Re-analysed on 2026-08-27_

This re-review resolves the two "did not exhaustively" hedges from the first pass by reading the callgraph neighbour `Mix.Utils.detect_user_id!/0` (the source of the predictable directory component) and by tracing the exact bytes written through `take_over`/`unlock`. Both hedges resolve to **verified caveats**, not hidden findings. All three findings survive unchanged. No file was modified.

## Summary

`Mix.Sync.Lock` reads, opens, and writes lock files by predictable name inside a shared-temp directory it does not verify it exclusively owns, so a local co-resident attacker can redirect the victim's lock writes through a symlink to corrupt an arbitrary victim-writable file and can crash any `mix` build by planting malformed lock-file bytes or a hostile lock peer.

## Security

#### `CSCAN-lock-symlink-write`
- **Finding ID** — `CSCAN-lock-symlink-write`
- **Severity** — Medium
- **Confidence** — high
- **Location** — lock.ex:421 `switch_file_replace!/2` (open/pwrite by name); root cause lock.ex:132-140 `base_path/0` + `lock/2` `File.mkdir_p!`
- **Class** — File operations and path handling (symlink following / insecure temp dir, CWE-59 + CWE-377)
- **Impact** — On a shared multi-user host (world-writable sticky `/tmp`), the lock directory path `<tmp>/mix_lock_v2_user<uid>/<md5>` is fully predictable and `File.mkdir_p!` reuses a pre-existing directory with no ownership or mode check. An attacker who wins the (persistent, since stale files are intentionally never cleaned) race to create that directory owns it and can plant a symlink at `lock_0`. `switch_file_replace!` opens `lock_0` with `File.open!(path, [:read, :write, :binary, :raw])` — which follows symlinks — so the victim's `take_over`/`unlock` writes lock-metadata bytes into the symlink target, corrupting or extending any file the victim uid can write (or crashing the build if the target's first byte is not `0`/`1`). This runs during `mix compile`/`deps.get`, i.e. the build path.
- **Evidence** — `verified`. Two independent facts confirmed:
  1. **Predictable, uid-derived path** — neighbour `Mix.Utils.detect_user_id!/0` (utils.ex:1023-1048) returns the OS uid read from `File.stat!(...).uid` (an integer, cached in `Mix.State`), so the `user<uid>` component is not a secret and `base_path/0` is fully determined by `System.tmp_dir!()` + uid. No random component exists in `base_path`; the only per-key component is the `md5(key)` hash, which is also predictable (the key is a build path / compiler identifier, not a secret).
  2. **End-to-end write redirection** — driving the real public API, the victim's `unlock` corrupted an attacker-redirected file:
  ```
  $ elixir /tmp/symlink_test.exs
  BEFORE: <<0, 79, 82, 73, 71, ...>>            # "ORIGINAL-IMPORTANT-CONTENTS"
  attacker swapped lock_0 -> symlink to /tmp/victim_important.dat
  AFTER:  <<1, 79, 82, ... 0, 0, 0, 0, ... 32, 0, ...>>   # byte0 toggled + 37 bytes pwritten at off 38
  ```
  Predictable path + no-ownership-check reuse confirmed: `base_path = /tmp/mix_lock_v2_user1000`, `/tmp` is `drwxrwxrwt`, and after an attacker `mkdir 0777` the victim's `File.mkdir_p!` left the mode `40777` untouched (`/tmp/tmpdir.exs`).
- **Suggested fix** — Create the per-user directory with mode `0700` and, if it already exists, `File.lstat` it and abort unless it is a real directory owned by the current uid with no group/other write; open/replace lock files in a way that refuses symlinks (`File.lstat` guard before `File.open!`, or an `O_NOFOLLOW`-equivalent raw open) rather than following them.

#### `CSCAN-lock-malformed-file-crash`
- **Finding ID** — `CSCAN-lock-malformed-file-crash`
- **Severity** — Low
- **Confidence** — high
- **Location** — lock.ex:446-457 `switch_file_read/1` and lock.ex:384-393 `decode_lock_info/1`
- **Class** — Deserialisation (non-defensive binary decoder → uncaught crash / DoS)
- **Impact** — Both decoders use hard binary matches and a non-total `case`: `<<switch_byte, rest::binary>> = data`, `<<c1::size(^n), c2::size(^n)>> = rest`, `case switch_byte do 0/1 end`, and `<<port::32, ps::8, _::binary-size(ps), os_pid::binary>> = data`. Any short, odd-length, or otherwise malformed lock file makes them raise `MatchError`/`CaseClauseError`, which propagates out of `with_lock/3` and aborts the build. Reachable whenever a malformed file sits at the predictable `lock_0` path (same local precondition as `CSCAN-lock-symlink-write`); normal operation never produces malformed content, so this fires only on tampering.
- **Evidence** — `verified`. End-to-end through the public API with a planted `lock_0`:
  ```
  $ elixir /tmp/e2e.exs
  planted malformed lock_0 = <<2, 255, 238, 221>>
  with_lock RAISED: MatchError - no match of right hand side value: <<255, 238, 221>>
  ```
  ("CRITICAL SECTION RAN" never printed.) Unit-level: empty/short/odd/`switch=2`/`switch=255` inputs all raise `MatchError`/`CaseClauseError` (`/tmp/decode_test.exs`).
- **Suggested fix** — Parse with `with`/`case` returning `{:error, :malformed}` on any non-conforming length or switch byte, and have `fetch_probe_port`/callers treat a malformed lock file as stale (replace/retry) instead of letting the exception escape.

#### `CSCAN-lock-await-close-caseclause`
- **Finding ID** — `CSCAN-lock-await-close-caseclause`
- **Severity** — Low
- **Confidence** — high
- **Location** — lock.ex:342-352 `await_close/1`
- **Class** — Network (non-exhaustive `case` on socket result → uncaught crash / DoS)
- **Impact** — `case recv(socket, 0) do {:error, :closed} -> ...; {:error, _other} -> ... end` has no clause for `{:ok, data}`. A hostile lock peer — whose port the attacker wrote into the lock file and which passes the 7-byte `@probe_data` check — can, instead of closing, send extra bytes; `recv(_, 0)` then returns `{:ok, data}`, raising `CaseClauseError` and crashing the waiting build. Same local precondition (control of lock-file content / a loopback port).
- **Evidence** — `verified`. `recv(socket, 0)` returns `{:ok, "extra-bytes-not-a-close"}` when the peer sends data, and the exact two-clause structure raises `CaseClauseError` on it (`/tmp/recv0.exs`).
- **Suggested fix** — Add an `{:ok, _data}` clause that closes and retries (or a catch-all treating unexpected data like an error), making the `case` total.

### Ruled out

- **Path traversal via `key`**
  - **Verdict** — ruled out. `key` is `:erlang.md5` + `Base.url_encode64`'d into a fixed hash before `Path.join`, so no external component reaches a path segment; the md5 use is a filename hash, not a security primitive, so collisions are harmless.
  - **Axes examined:** hash encoding (url_encode64 emits only `[A-Za-z0-9_-]`, no `/` or `.`), key content reaching a path segment. **Not examined:** none — the only `Path.join` component derived from external input is the fixed-width base64 of an md5 digest, which structurally cannot contain a path separator.

- **Term-injection via `decode_lock_info`**
  - **Verdict** — ruled out. It only extracts an integer `port` and a binary `os_pid`; no atoms, funs, or `binary_to_term`, and the decoded `os_pid` flows only to the user `on_taken` callback (lock.ex:193), never back into `encode_lock_info` (which always re-encodes the *current* process's pid — see the content-provenance note below).
  - **Axes examined:** decoder output types (integer + binary only), downstream use of `os_pid` (traced to `on_taken.(os_pid)` and nowhere else). **Not examined:** none — grep of `os_pid`/`decode_lock_info` uses confirms the decoded value never reaches an atom/term/eval sink.

- **`port_P` symlink write**
  - **Verdict** — ruled out. `try_lock` calls `file_rm_if_exists!(port_path)` (lock.ex:181), which unlinks a planted symlink *itself* (not its target), before `switch_file_create!` writes via `File.write!`. So a planted `port_*` symlink cannot redirect that write.
  - **Axes examined:** `File.rm` on a symlink (removes the link, verified in the escript-uninstall run recorded in memory `[[mix-escript-uninstall-review]]`), ordering of `file_rm_if_exists!` before `switch_file_create!`. **Not examined:** none for the `port_P` path — the unlink-then-create sequence removes any pre-planted entry. (The residual symlink risk lives entirely at `lock_0` via `switch_file_replace!`, which is the separate `CSCAN-lock-symlink-write` finding; `switch_file_create!` uses `File.write!` with truncation, not `File.open!` on an existing path.)

- **Remote reachability**
  - **Verdict** — ruled out. All sockets bind/connect `@loopback = {127,0,0,1}` (lock.ex:69-71) and every port connected to is read from a local lock file; no off-host peer can reach the lock ports, so every finding here is local-only.
  - **Axes examined:** listen/connect address (hardcoded loopback), source of connect ports (local file). **Not examined:** none — the address is a compile-time constant with no override path.

- **Attacker-controlled *content* of the symlink write (escalation of `CSCAN-lock-symlink-write`)**
  - **Verdict** — ruled out as an escalation vector. The bytes written through the redirected `lock_0` handle are `encode_lock_info(port, os_pid)` in `take_over` (lock.ex:326) and `encode_lock_info(0, "")` in `unlock` (lock.ex:357). In `take_over`, `port` is the victim's own TCP port and `os_pid = System.pid()` is the victim's own OS pid (set at lock.ex:175 and threaded through `try_lock`→`take_over`); in `unlock` the content is a fixed `port=0, os_pid=""`. None of these are attacker-influenceable, so the primitive corrupts the target with the victim's own port/pid metadata and zero-padding, plus a toggled switch byte — an integrity/availability primitive, not an attacker-chosen-content write. `File.open!([:read, :write])` also requires the target to already exist (it does not create), so it cannot forge new files.
  - **Axes examined:** provenance of every argument to the two `encode_lock_info` call sites feeding `switch_file_replace!` (both traced to victim-local values), file-creation semantics of `[:read, :write]` open. **Not examined:** none — the content arguments are fully determined by the victim's own process state.

## Did I verify this

- **`base_path` predictability (neighbour read).** Verified by reading `Mix.Utils.detect_user_id!/0` (utils.ex:1023-1048): the `user<uid>` component is the integer OS uid from `File.stat!`, cached in `Mix.State`, with no random or secret element. Confirms the Medium finding's "fully predictable path" premise rather than assuming it.
- **Hedge 1 — "did not attempt to escalate to content-controlled writes."** Resolved to a **verified ruled-out** entry (above): I traced both write-content sources (`take_over` and `unlock`) and confirmed the bytes are the victim's own port/pid, so the primitive is integrity/availability-only. This is a code-path provenance fact (established by reading the two call sites), not an axis-disagreement, so a read suffices to close it. The Medium rating stands unchanged (unchanged content-control limitation was already the basis for Medium, not High).
- **Hedge 2 — "did not exhaustively drive the full chained take-over protocol for mutual-exclusion breaks."** The attacker-reachable surfaces of the `grab_lock`/`take_over`/`await_close` protocol are exactly the untrusted-input sinks already captured by `CSCAN-lock-malformed-file-crash` (planted file bytes) and `CSCAN-lock-await-close-caseclause` (hostile peer bytes), both verified end-to-end. The residual concern — whether two *honest* concurrent Mix processes can both believe they hold the lock — is a correctness/mutual-exclusion property with no external-input boundary crossing (both parties are the same trusted uid running `mix`), so it falls outside the untrusted-input security rubric and is not upgraded. I did not fuzz N-way concurrent honest contention; I state this as an out-of-scope correctness caveat rather than a hedged security assumption.
- **All three findings:** executed end-to-end through the public `with_lock/3` API on this machine (OTP/Elixir toolchain present); evidence transcripts cited inline. No re-run was needed to preserve them.

## Risks / caveats

- All three findings share one precondition — a local attacker able to write the victim's predictable lock directory — realistic only on shared multi-user hosts with a shared world-writable temp (typical build/CI servers), not on single-user machines or per-job-isolated CI. That is why the symlink write is Medium and the two crashes Low despite verified mechanisms.
- The symlink primitive is integrity/availability-only (victim's own port/pid + zero bytes, target must pre-exist); this is now recorded as a verified ruled-out escalation vector rather than an open question.
- Mutual-exclusion correctness of the chained take-over algorithm under concurrent *honest* processes is a correctness concern outside the untrusted-input security scope; not fuzzed, explicitly out of rubric.
- A linear slowdown is possible if an attacker plants many `lock_N` files (bounded by attacker effort), noted but not rated.

### Dropped in re-review

None. All three findings and all four original ruled-out entries are preserved; one additional ruled-out entry (content-controlled-write escalation) was added to convert a first-pass caveat into an explicit verified dismissal.
