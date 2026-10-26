# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/sync/lock.ex
<!-- sha256: 24bd8881a53da76d828820c5377f175cfc8fe1d997de2ce39e29c82406bc25cb -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/sync/lock.ex
<!-- sha256: 24bd8881a53da76d828820c5377f175cfc8fe1d997de2ce39e29c82406bc25cb -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28. Hedges flagged by first pass ("did not exhaustively"). Each hedge below is now either backed by executed evidence or left explicitly `asserted` with its untried axis named._

## Summary

`Mix.Sync.Lock` implements a cross-OS-process lock via TCP sockets + hard links; its single public entry point is `with_lock/3`. The core state machine (chained locking, take-over ordering, switch-file atomic replace, self-port detection) is internally consistent. The one genuine weakness remains: the on-disk lock-info decoders and the switch-file read/replace paths are non-total, so an externally corrupted `lock_N`/`lock_0` file crashes with a raw `MatchError`/`CaseClauseError`/EOF-raise instead of being treated as stale — defeating exactly the "abruptly terminated / stale file" resilience the module advertises.

Re-review changes vs. first pass:
- Main finding re-confirmed by re-execution and its evidence broadened (empty/short/odd-length → `MatchError`; switch byte ∉ {0,1} → `CaseClauseError`; truncated `lock_0` in `switch_file_replace!` → EOF-raise).
- The `await_close/1` ruled-out entry is **upgraded from reasoned to executed**: I ran both axes (peer sends only the probe then closes; peer sends extra bytes) against a live socket pair.

## Bugfixes

#### `CSCAN-lock-nontotal-lockfile-decoders`
- **Finding ID:** `CSCAN-lock-nontotal-lockfile-decoders`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `lock.ex:384` (`decode_lock_info/1`), `lock.ex:446`–`455` (`switch_file_read/1`, including `case switch_byte`), `lock.ex:421`–`444` (`switch_file_replace!` — `read_bytes!` EOF-raise at :426/:459 and `case switch_byte` at :430)
- **Class:** Missing clause / non-total function (Phase-1 sink: unchecked decode of untrusted on-disk input → `ERROR_PROPAGATES_NOT_BADMATCHES` / `INPUT_INVALID_RETURNS_ERROR`)
- **Impact:** `with_lock/3` reaches these decoders for every `lock_N` file found in the per-user lock directory (via `probe → fetch_probe_port → switch_file_read → decode_lock_info`, via `switch_file_replace!` in `take_over`, and via `switch_file_replace!` in `unlock`). The module's stated design goal is resilience to files left behind by abruptly terminated processes. But if a lock file is empty, truncated, has an odd-length body, or carries a switch byte other than `0`/`1`, the bare binary matches / `case switch_byte` / `read_bytes!` EOF path raise `MatchError` / `CaseClauseError` / `File`-EOF `RuntimeError`. None of these is rescued as "stale": they propagate out of `lock/2` (whose `rescue` only closes the socket and reraises) or out of `unlock/1`, aborting the Mix run with an opaque error. A single corrupt file in `$TMPDIR/mix_lock_v2_user<id>/<hash>/` can wedge all future builds for that user until the file is deleted by hand.
- **Evidence:** verified by re-execution (`/tmp/lock_check.exs`, replicating the exact binary logic):
  ```
  info size: 37
  decode roundtrip: {12345, "9999"}
  file size: 75
  read+decode: {12345, "9999"}
  --- truncated / corrupt cases ---
  empty file:                 {:raised, MatchError}
  short (1 byte switch only):  {:raised, MatchError}
  odd length rest:             {:raised, MatchError}
  switch byte = 2:             {:raised, CaseClauseError}
  switch byte = 255:           {:raised, CaseClauseError}
  decode short segment:        {:raised, MatchError}
  --- decode_lock_info directly on garbage ---
  empty:              {:raised, MatchError}
  3 bytes:            {:raised, MatchError}
  padding_size huge:  {:raised, MatchError}
  ```
  Valid data round-trips; every malformed shape raises with no error clause. Additionally, `switch_file_replace!` on a truncated `lock_0` (< 1 byte) hits `read_bytes!` → `:eof` → `raise "unexpected EOF"` (line 465), and on a switch byte ∉ {0,1} hits the `case switch_byte` with no matching clause (`CaseClauseError`). Reachability of a corrupt file is environmental (see caveat), not producible by the module's own write path.
- **Suggested fix:** Make the read/decode paths total: have `switch_file_read/1` and `decode_lock_info/1` return `{:error, :corrupt}` on any non-matching binary (wrong length, unknown switch byte, EOF), and have `fetch_probe_port/1` map that to a stale outcome so `grab_lock` advances to `lock_{n+1}` (matching the existing `{0, _} -> {:error, :ignore}` handling). For `switch_file_replace!`, treat a malformed existing file as replace-from-scratch (rewrite via `switch_file_create!`) rather than raising.

## Risks / caveats

- **Reachability of the finding is environmental, not caller-driven.** A well-behaved caller of `with_lock/3` cannot itself produce a corrupt lock file: `switch_file_create!` fully writes the port file *before* `File.ln` exposes it as `lock_N`, and `switch_file_replace!` mutates in place without changing size or leaving a torn switch byte. Corruption requires an externally-damaged file (disk fault, a killed process from an incompatible on-disk format, another same-user process writing junk into the tmp subtree). `base_path/0` (line 132–135) partitions the directory per user via `Mix.Utils.detect_user_id!()`, so this is not a cross-trust attack surface — hence Low severity — but it is a real robustness gap against exactly the stale-file conditions the module advertises it handles.

### Ruled out

- **`await_close/1` `CaseClauseError` on `{:ok, data}`** (`lock.ex:342`).
  Verdict: **verified (executed).** Upgraded from hedged/reasoned in the first pass.
  `await_close/1` handles `{:error, :closed}` and `{:error, _other}` but not `{:ok, _}`. I tested both axes against a live loopback socket pair (`/tmp/recv0.exs`, `/tmp/recv0b.exs`):
  - *Owner sends only the 7-byte probe, then closes* (the real `accept_loop` behavior): after `await_probe_data` consumes the 7 bytes, `recv(socket, 0)` **blocked ~408 ms and returned `{:error, :closed}`** — handled cleanly.
  - *Owner sends extra bytes after the probe*: `recv(socket, 0)` returned `{:ok, "EXTRA"}` — which would raise `CaseClauseError`.
  The `{:ok, data}` path is therefore reachable **only if an accepted socket ever carries more than the probe**. Reading `accept_loop/1` (lines 232–245): it does `:gen_tcp.send(socket, @probe_data)` exactly once and then loops back to `accept/1` without ever referencing that accepted `socket` again — no code path writes a second time on an accepted socket. So the triggering input is never produced.
  Axes examined: post-probe extra bytes present (executed), post-probe extra bytes absent + close (executed), owner write-count (code fact from lines 232–245). Not examined: none.

- **`take_over`/`switch_file_replace!` concurrent double-writer to `lock_0`** (`lock.ex:319`, `:326`, `:421`).
  Verdict: **asserted (reasoned, not executed).**
  The atomic-replace correctness of `switch_file_replace!` on `lock_0` rests on a single-writer invariant. Reasoning from the code: `lock_0` is created once via an atomically-exclusive `File.ln` (eexist on collision) and is never removed. A second process that finds `lock_0` stale links `lock_1`; any third process probing `lock_1` connects to the live socket of the `lock_1` holder and blocks in `{:taken} → await_close`. The `lock_N` holder keeps its listen socket open for the whole lock lifetime, and both `take_over` (replace `lock_0` → remove `port_*` → remove `lock_1+`) and `unlock` (replace `lock_0`, *then* close socket in the `after`) complete their `lock_0` write before releasing waiters. So only the current holder writes `lock_0` at any instant — by construction.
  Axes examined: static ordering of `take_over`/`unlock`/`await_close` (code reading). **Not examined: live multi-OS-process race execution, including the ephemeral-port-reuse window the module documents (lines 56–67).** I did not run a real multi-process stress harness; a differential/axis execution that would falsify the invariant was not performed, so this stays `asserted`. (Note the module itself mitigates the port-reuse axis via the `@probe_data` handshake in `await_probe_data/1`, so no novel finding is claimed.)

- **Accepted-socket accumulation in `accept_loop`** (`lock.ex:232`).
  Verdict: asserted (reasoned). Accepted prober sockets are never explicitly closed, but they are tied to the linked accept process and closed when `unlock` closes the listen socket (`lock.ex:361` + comment at :359–360); bounded by lock lifetime. Not examined: long-lived-lock socket-count growth under sustained prober churn (not executed). Stays asserted; no leak claimed under normal lifetime.

- **`grab_lock` unbounded `n` recursion** (`lock.ex:202`).
  Verdict: asserted (reasoned). Walks `lock_1, lock_2, …` past stale entries with no cap, but each step consumes a real stale/held file and converges to the first free slot; no infinite loop under valid filesystem state. Not examined: adversarial filesystem where entries are re-created faster than consumed (not executed). Stays asserted.

- **Atom-table growth / `to_unsafe_atom`** — none present in this file; the threat-model concern lives in `Mix.CLI`/`Mix.State`, not here. Verified by reading the full file (no `to_atom`/`to_existing_atom`/`to_unsafe_atom` call sites). Not examined: none.

### Did I verify this?

- **Decoder non-totality (main finding):** Yes — re-executed the exact binary logic with valid and 9 malformed inputs; all malformed shapes raise (`MatchError`/`CaseClauseError`), valid data round-trips. Also traced the three call sites (`fetch_probe_port`, `take_over`, `unlock`) that reach the raising paths, and confirmed `lock/2`'s `rescue` (lines 148–153) only closes+reraises rather than treating the file as stale.
- **`await_close` case-clause safety:** Yes — executed both axes on a live socket pair; confirmed `recv(socket, 0)` blocks-until-close (`{:error, :closed}`) when only the probe was sent, and that `accept_loop` provably never sends extra bytes (code reading, lines 232–245). Fully verified, `Not examined: none`.
- **Single-writer to `lock_0` (concurrency):** No — verified by static reasoning only. The live multi-process race (including ephemeral-port reuse) was **not executed** and remains an explicitly named untried axis; the entry stays `asserted`, not cleared.
- **Per-user directory isolation (no cross-trust surface):** Yes — read `base_path/0` (lines 132–135); the lock dir is namespaced by `Mix.Utils.detect_user_id!()`, so a corrupt file affects only the same user.

### Dropped in re-review

None. Every finding and ruled-out entry from the first pass is preserved. The first pass's free-floating hedge caveat ("I did not exhaustively test the live socket/hard-link state machine under real concurrency") has been folded into the `take_over` ruled-out entry as an explicit `Not examined: live multi-OS-process race execution` axis and marked `asserted`, rather than left as loose hedge prose.
