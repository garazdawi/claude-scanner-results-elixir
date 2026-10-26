# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.partition.ex
<!-- sha256: 7eec7a7f57151c75e34351f76531428f4f872502d8fc6b20e48ea2ca51583fb6 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.partition.ex
<!-- sha256: 7eec7a7f57151c75e34351f76531428f4f872502d8fc6b20e48ea2ca51583fb6 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (hedge-resolution pass)_

## Summary

`deps.partition.ex` is the internal driver that fans dependency compilation out across worker OS processes over a TCP control channel. The compile logic is sound, but the failure-recovery path (`tcp_failed!/4`) destructures a `pop_with/2` result that is `:error` whenever the failing socket belonged to an **idle** worker (in `available`, not `busy`), turning a recoverable worker death into an opaque `MatchError` that also skips socket/port cleanup. The per-run config temp file written under `$TMPDIR/mix_deps_partition/` is never deleted on any path. Both findings are retained; evidence for both was strengthened this pass (repro re-run for the crash; tree-wide grep for the leak). Two "outside this file" caveats from the first pass are now resolved by reading neighbour modules: `Mix.Tasks.Deps.Compile.compile_single/3` returns a strict boolean, so the `"#{app}:#{compiled?}"` message round-trip carries only `true`/`false`.

## Bugfixes

#### `CSCAN-deps.partition-tcp-failed-available-socket`
- **Finding ID:** `CSCAN-deps.partition-tcp-failed-available-socket`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `elixir/lib/mix/lib/mix/tasks/deps.partition.ex:193` (`tcp_failed!/4`), reached from `server_loop/4` at lines 168–172.
- **Class:** Missing clause / non-total function (destructure of a value that can be `:error`) → `ERROR_PROPAGATES_NOT_BADMATCHES`.
- **Impact:** When a partition worker's socket closes or errors while that worker is **idle** (sitting in the `available` list, not `busy`), `server_loop` (lines 168/171) forwards the socket to `tcp_failed!/4`. There, `pop_with(busy, &(&1.socket == socket))` (line 193) returns `:error` because the socket is not in `busy`, and the head `{%{port: port} = client, busy} = pop_with(...)` raises `MatchError`. The user gets `** (MatchError) no match of right hand side value: :error` instead of the intended diagnostic `mix deps.partition ... closed unexpectedly (set MIX_OS_DEPS_COMPILE_PARTITION_COUNT=1 to run in serial)`. Because the raise happens *before* `shutdown_clients(available ++ busy ++ [client])` (line 204), the remaining worker sockets/ports are not torn down on this exit path. Idle workers are common early in a build: sockets are `active: true` (line 101), so `{:tcp_closed, _}`/`{:tcp_error, _}` is delivered for any worker socket, and with `MIX_OS_DEPS_COMPILE_PARTITION_COUNT` larger than the number of currently-schedulable (leaf) deps, several workers sit in `available`; one dying (OOM-kill, crash) in that window hits this path.
- **Evidence:** verified — the defective destructure was re-run this pass against a `busy` list not containing the socket:
  ```
  $ elixir repro.exs
  RAISED: MatchError -> no match of right hand side value:
      :error
  ```
  `pop_with/2` (lines 185–190) returns `:error` whenever no element matches (`Enum.split_while(busy, &(not fun.(&1)))` yields `{busy, []}` when the socket is absent from `busy`). Reachability of the idle-socket case rests on code-evident facts (line 101 `active: true`; `send_deps` leaves surplus clients in `available`; `server_loop`'s `{:tcp_closed, socket}` clause routes *any* socket to `tcp_failed!`), not on a stood-up multi-worker harness — the destructure crash itself is executed and confirmed.
- **Suggested fix:** Search both queues before failing, e.g. `Enum.find(busy ++ available, &(&1.socket == socket))`, remove it from whichever list holds it, and preserve the `Mix.raise` diagnostic and `shutdown_clients` cleanup on the idle-worker-death path.

#### `CSCAN-deps.partition-config-tempfile-leak`
- **Finding ID:** `CSCAN-deps.partition-config-tempfile-leak`
- **Severity:** Low
- **Confidence:** high  *(raised from medium: leak existence now verified tree-wide)*
- **Location:** `elixir/lib/mix/lib/mix/tasks/deps.partition.ex:41-50` (`server/4`), config-file branch.
- **Class:** Resource leak (temp file) → `RESOURCE_CLEANED_ON_ERROR_PATH`.
- **Impact:** Each `server/4` invocation writes the serialized project config to `System.tmp_dir()/mix_deps_partition/<random>` (lines 43–46) and passes it to workers via `--config-file`. Nothing ever deletes that file or the `mix_deps_partition` directory — not on success, not on error, and not in the worker, which only `File.read!`s it (`client/0`, line 266). Every partitioned `mix deps.compile` run leaves one small file behind in the shared temp dir, accumulating until the OS reaps `$TMPDIR`. Bounded and small, so impact is minor, but it is an unbounded-over-time leak on long-lived dev machines / persistent CI caches.
- **Evidence:** verified — `grep mix_deps_partition` across the entire `elixir/lib/mix` tree returns exactly one hit, the `Path.join` at line 43; there is no `File.rm`/`File.rm_rf`/`File.rm_rf!` of `partition_file` or `partition_dir` anywhere in the tree. Confirmed the worker path only reads the file (line 266).
- **Suggested fix:** Delete `partition_file` after all workers have connected and read it (before returning from `server/4`, tracking each file once its worker consumes it), or write into a per-run subdirectory removed with `File.rm_rf` in an outer `after`.

## Risks / caveats

- This module is `@moduledoc false` (internal to Mix's `deps.compile` machinery); its `def`s are exported and were reviewed as the file's reviewable surface. A "documented public API only" purist could discount both findings on the grounds that only Mix itself calls them; I treat the exported functions as the surface.
- The `MatchError` finding's *defect* is executed and confirmed; its *reachability* for the idle-socket case is established by code reading (active sockets + surplus-client scheduling + the `{:tcp_closed, socket}` clause), not by an end-to-end multi-worker harness. I judged a harness unnecessary because every intermediate step is unconditional in the source.
- **Resolved from first pass ("outside this file"):** `compile_single/3` (read this pass, `deps.compile.ex:122–181`) returns a strict boolean on every branch (`do_*` results, or explicit `false`/`true`). Therefore the worker's `"#{app}:#{compiled?}\n"` (line 322) always carries `"true"`/`"false"`, and the server's `String.to_unsafe_atom` on that field yields the atoms `true`/`false` — matching the `elem(&1,1) == true` check at line 149. No defect in the compiled-flag round-trip.
- **Resolved from first pass ("outside this file"):** `Base.url_encode64`/`:erlang.term_to_binary` (server, lines 36–37) and their inverses `Base.url_decode64!`/`:erlang.binary_to_term` (worker, lines 270–271) are symmetric stdlib operations over a term the server itself produced; no asymmetry to exploit and no finding. `Mix.CLI.main(["deps.partition" | ...], nil)` re-entry (line 289) dispatches back into this module's `run/1`; benign, no state defect introduced here.

### Ruled out

- **`Enum.any?(status, &(elem(&1,1) == true))` mishandling the `:true` atom** (line 149).
  - *Axes examined:* value round-trip — re-ran `String.to_atom("true") == true` → `true` this pass (`:true` is the boolean); and the *source* of `compiled?` — `compile_single/3` verified to return only `true`/`false`.
  - *Not examined:* a **misbehaving/spoofed worker** sending a `compiled?` field that is neither `"true"` nor `"false"` (would require a peer other than this module's own `client_loop`). Untried — stays asserted for that axis.
  - Verdict: not a bug for valid inputs.

- **`[app, compiled?] = data |> String.split(":") |> Enum.map(&String.to_unsafe_atom/1)` crashing / atom growth** (lines 155–156).
  - *Axes examined:* colon-count for well-formed messages — the only producer is this module's `client_loop` emitting `"#{app}:#{compiled?}"` where `app` is an OTP application atom (no `:`) and `compiled?` is boolean, so exactly two parts; verified `compile_single` keeps `compiled?` boolean.
  - *Not examined:* (a) a message with an unexpected number of `:`-separated segments from a non-conforming peer → `MatchError` on the two-element destructure; (b) **`String.to_unsafe_atom` on attacker-chosen segments → atom-table growth** if a local process connects to the ephemeral listen port (`:gen_tcp.listen(0, ...)`) before the real workers and sends crafted lines. Both require a peer other than the trusted worker; the server accepts only `count` connections and workers connect immediately, so this is a pre-existing localhost-race design property, not exercised by valid inputs. Untried — these axes stay asserted.
  - Verdict: not reachable via valid inputs; the atom-from-network axis noted as a low-confidence residual, not upgraded (no executed evidence of a winnable race).

- **`server_loop` never matching `{port, {:exit_status, n}}`** — those messages linger in the mailbox but are drained by `loop_close_port/2` (lines 233–241) during `close_port`/`shutdown_clients`, and are harmless at process exit.
  - *Axes examined:* message drain path — read `loop_close_port` this pass; it explicitly matches `{^port, {:exit_status, _}}` and recurses.
  - *Not examined:* none.
  - Verdict: no behavioral defect.

- **Dependency-cycle deadlock in `server_loop`** (all remaining deps have a `:pending` sub-dep, no busy workers → blocks in `receive` forever).
  - *Axes examined:* whether cycles can reach partitioning — Mix rejects dependency cycles before partitioning, so `deps` handed here is acyclic.
  - *Not examined:* none reachable through valid Mix flow.
  - Verdict: not reachable.

## Did I verify this?

- **`tcp_failed!` MatchError:** Yes — re-executed the `pop_with(busy, ...) → :error` destructure (`elixir repro.exs`) and observed the `MatchError`. Reachability of the idle case established by reading lines 101, 126–145, 168–172, 192–204.
- **Temp-file leak:** Yes — tree-wide `grep mix_deps_partition` (one hit, no deletion) plus reading the worker read path (line 266). Upgraded evidence from `asserted` to `verified`.
- **`compiled?` boolean round-trip / message format:** Yes — read `compile_single/3` (`deps.compile.ex:122–181`), confirmed strict boolean return; re-ran the `:true == true` identity.
- **Encode/decode symmetry and `Mix.CLI.main` re-entry:** Reasoned from stdlib symmetry and the dispatch target; not independently executed (benign, no finding).

### Dropped in re-review

None. Both prior findings are retained (one with strengthened evidence and a raised confidence). No prior finding was withdrawn.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 2 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`[app, compiled?] = ... String.split(":")` crashing on malformed messages** — messages are produced by this module's own `client_loop` as `"#{app}:#{compiled?}"` with boolean `compiled?`; a well-behaved worker never yields other than two parts. Not reachable via valid inputs.

- **`Enum.any?(status, &(elem(&1,1) == true))` treating the `:true` atom wrong** — `String.to_unsafe_atom("true")` is the boolean `true`, so `:true == true` holds; verified this is identity in Elixir. Not a bug.
