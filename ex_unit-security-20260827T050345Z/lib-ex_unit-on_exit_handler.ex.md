# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/on_exit_handler.ex
<!-- sha256: e39b79916a4e46c53a1c83e6dd497912cf785ac0c03260d7f571d400495d8e8a -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/on_exit_handler.ex
<!-- sha256: e39b79916a4e46c53a1c83e6dd497912cf785ac0c03260d7f571d400495d8e8a -->

_Re-reviewed on 2026-08-27_

`ExUnit.OnExitHandler` is an internal (`@moduledoc false`) test-runner module that records per-test `on_exit` cleanup callbacks in the shared `ExUnit.Server` ETS table and executes them in a monitored, timeout-bounded runner process. The only sinks are ETS operations on the `ExUnit.Server` table, process-lifecycle calls (`monitor`/`exit`/`spawn_monitor`), message passing, and invocation of the developer's own `on_exit` callback (`callback.()` / `exec_callback(fun)`). This pass resolves the two cross-file hedges from the first pass by reading `server.ex` and every caller of this module.

**Cross-file verification performed this pass:**
- **ETS table creation.** `ExUnit.Server.init/1` (`server.ex:57–59`) calls `:ets.new(__MODULE__, [:public, :named_table, read_concurrency: true, write_concurrency: true])`. The table name is `ExUnit.Server`, matching `@name ExUnit.Server` here (line 9). Confirmed by reading the file, not assumed: the table is created `:public` and `:named_table` by the owning server before any handler touches it. The first-pass "I assumed … `ExUnit.Server.init/1` creates the table" hedge is now a read fact.
- **Callers of this module.** Grep for `OnExitHandler.(register|add|get_supervisor|put_supervisor|run)` across `elixir/lib/ex_unit` returns exactly three trusted internal call sites: `ex_unit.ex:547/550`, `runner.ex:353/449/541`, and `callbacks.ex:514/663`. No caller outside ExUnit's own internals invokes these functions. The first-pass "I assume … only trusted ExUnit internals call these functions" hedge is now enumerated and verified.
- **`name_or_ref` provenance.** `callbacks.ex:513` defines `def on_exit(name_or_ref \\ make_ref(), callback) …` and forwards it verbatim to `OnExitHandler.add/3` at line 514. The value is either an auto `make_ref()` or a developer-chosen term from the test author's own `on_exit/2` call — trusted developer input, never externally derived. Within this file it is stored only as the `List.keystore` key (line 29); it is never passed to `String.to_atom`, `Module.concat`, `apply`, a path, or `binary_to_term`. Confirmed by reading both files.

No `String.to_atom`, `binary_to_term`, `Code.eval`, `Path`/`File`, `System.cmd`/`:os.cmd`, `Regex`, network, or format-string sink exists in the file. There is no path by which untrusted data reaches any sink here through a documented public API.

## Summary

`ExUnit.OnExitHandler` records developer-authored `on_exit` callbacks in the `ExUnit.Server` ETS table and runs them in a monitored, timeout-bounded process. Its entire input surface is trusted internal pids and developer callbacks; every cross-file dependency the first pass had hedged (ETS-table ownership, caller set, `name_or_ref` provenance) is now verified by reading `server.ex`, `callbacks.ex`, `runner.ex`, and `ex_unit.ex`. I found no security defect reachable from untrusted data.

## Security

No findings.

### Ruled out

- **`callback.()` / `exec_callback(fun)` arbitrary code execution (`exec_callback/1`, line 140; `on_exit_runner_loop/0`, line 126).** Confidence: high. The executed function is the developer's own `on_exit` block, forwarded verbatim from `ExUnit.Callbacks.on_exit/2` (`callbacks.ex:513–514`, verified this pass). The test author already has full code execution; running their own callback is not a trust crossing. No external byte stream is compiled or evaluated (no `Code.eval*`, no quoted-AST unquoting). Axes examined: input provenance (traced to `on_exit/2` default `make_ref()` or developer term), sink type (bare closure `.()`), evaluation of external bytes (none). Not examined: none.

- **Atom creation from `name_or_ref` (`add/3`, line 29).** Confidence: high. `name_or_ref` is stored as-is as the key of a `List.keystore` tuple; it is never passed to `String.to_atom`/`Module.concat`/`apply`. Grep across this file and its sole producer (`callbacks.ex:513`) confirms the value is a `make_ref()` or developer term and reaches no atom/dispatch sink. Axes examined: every use of `name_or_ref` in-file (single keystore key), producer call site. Not examined: none.

- **Public ETS table poisoning (`@name ExUnit.Server`, lines 17/24/30/38/46/54).** Confidence: high. The table is created `:public` in `ExUnit.Server.init/1` (`server.ex:59`, read this pass), so a co-located process could corrupt entries, but (a) creation and ownership live in `ExUnit.Server`, not this file, and (b) writing it requires a process already running on the node — the co-tenant/local-code-exec model the folder threat model places out of scope. All keys/values written from this file are trusted internal pids. Axes examined: table owner (server.ex init), visibility flag (`:public`, confirmed), writers in this file (all internal pids). Not examined: none.

- **`add/3` lookup→update TOCTOU (lines 24–30).** Confidence: high. `lookup_element` then `update_element` are separate operations; concurrent `add` for the same registered pid could clobber a callback list. The racers are trusted test processes (the callback is added by `self()` in `callbacks.ex:514`, i.e. the single test process registering sequentially), and the worst outcome is a skipped cleanup callback — a test-isolation/correctness concern, not a security one, with no untrusted data involved. Axes examined: racer identity (test process via `self()`), data trust (internal pids/developer callbacks), impact class (isolation, not confidentiality/integrity of external data). Not examined: none.

- **Stale reply left in mailbox after a timeout-kill (`receive_runner_reply/4`, lines 94–118).** Confidence: high. If a runner replies just before being `:kill`ed, its `{runner_pid, reason}` message can linger in the mailbox (a new runner is spawned with a fresh pid). This is a benign, bounded leak inside a trusted process about to exit; no security impact. Axes examined: message shape, process lifecycle, boundedness (one stale term per killed runner). Not examined: none.

- **`nil ->` re-loop on already-dead runner (lines 114–115).** Confidence: high. `Process.info/2` returns `nil` only when the monitored process is dead, which guarantees a `:DOWN` is (or will be) in the mailbox, so the re-entered `receive` returns via the DOWN clause rather than hanging. Not a DoS. Axes examined: `Process.info` nil semantics, monitor delivery guarantee. Not examined: none.

- **Dead defensive code check.** Confidence: high. The `try/rescue` in `add/3` (line 25) and `get_supervisor/1` (line 39) are reachable when the pid is absent from the table (e.g. called after `run/2`'s `:ets.take`), returning `:error`; the `terminate_supervisor(nil, …)` clause (line 59) and the timeout branches are all reachable. No guard here is unreachable. Axes examined: each rescue/branch reachability. Not examined: none.

## Risks / caveats

- **Reproduction not applicable.** No sink in this file accepts data crossing a trust boundary, so there is no hostile input to drive; the "No findings" conclusion rests on trust-tracing — now anchored on read facts for the cross-file links — rather than a behavioural repro, which is the appropriate outcome for pure internal plumbing.
- **Cross-file dependency (verified).** `ExUnit.Server.init/1` creates the `ExUnit.Server` ETS table `:public`/`:named_table` before any handler runs (`server.ex:57–59`), and the only callers of this module are `ex_unit.ex`, `runner.ex`, and `callbacks.ex` — all ExUnit internals (grep-enumerated this pass). `name_or_ref` originates from `on_exit/2`'s `make_ref()` default or a developer term (`callbacks.ex:513`) and is never forwarded into an atom/dispatch sink in this file. If some future caller were to forward externally-derived values as `name_or_ref` into a downstream sink, that risk would live in that caller; this file never converts `name_or_ref`.
- **Co-tenant model.** The `:public` ETS table and the exported `on_exit_runner_loop/0` are writable/messageable by any process on the same BEAM node. This is only exploitable by an actor who already has local code execution, which the threat model treats as out of scope; the mitigation (table visibility) belongs to `ExUnit.Server`, not this module.

## Did I verify this?

- ETS table ownership/visibility: **verified** — read `server.ex:57–59`, table created `:public`/`:named_table` by the owning GenServer's `init/1`.
- Caller set / `name_or_ref` provenance: **verified** — grep enumerated all five call sites across three internal modules; read `callbacks.ex:513–514` confirming `name_or_ref` is `make_ref()`/developer term forwarded verbatim.
- In-file sink inventory (no atom/eval/path/btt/format-string sinks): **verified** by full read of the file.
- Behavioural repro: **not applicable** — no trust-boundary-crossing input exists to drive; no axis here requires a dual-implementation comparison, so nothing was left on a `Not examined` list.

## Dropped in re-review

None. Every prior ruled-out entry is retained. The two first-pass hedges ("I assumed … `ExUnit.Server.init/1` creates the table"; "I assume … only trusted ExUnit internals call these functions") were not findings to drop but unverified assumptions supporting the "No findings" conclusion; both are now verified by reading `server.ex` and enumerating callers, so no assumption needed upgrading to a low-confidence finding.
