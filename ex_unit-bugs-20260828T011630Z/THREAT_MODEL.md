I have enough context to write the summary.

# Folder Correctness/Bug Profile — `elixir/lib/ex_unit`

## Protocol / standard implemented
This is **ExUnit**, Elixir's built-in unit-testing framework (part of the Elixir standard library, Apache-2.0, Elixir Team / Plataformatec). There is no wire protocol or RFC — the governing "spec" is ExUnit's own public API contract and OTP behaviours (`GenServer`, `Supervisor`, `DynamicSupervisor`, `:gen_event`, `:logger` handler API). Correctness here means *test-lifecycle fidelity*: every test runs, its result is reported exactly once, callbacks fire in order, and captured I/O/logs are reliably restored. Bugs manifest not as security breaches but as **mis-reported test outcomes, hangs, leaked global process/registration state, and lost captured output** — silent corruption of a test suite's trustworthiness.

Documented public entry points: `ExUnit` (`start/1`, `run/0`, `configure/1`, `after_suite/1`), `ExUnit.Case`, `ExUnit.Callbacks` (`setup`, `setup_all`, `on_exit`, `start_supervised`), `ExUnit.Assertions` (`assert`, `refute`, `assert_receive`, …), `ExUnit.CaptureIO`, `ExUnit.CaptureLog`, `ExUnit.DocTest`, `ExUnit.Filters`, `ExUnit.Formatter`. Most modules here (`Server`, `Runner`, `RunnerStats`, `CaptureServer`, `EventManager`, `OnExitHandler`) are `@moduledoc false` internals.

## Trust / contract boundaries
The threat model is **not adversarial** — inputs come from test authors and configuration, not network peers. Boundaries:
- **Operator/config-supplied**: seed, `max_cases`, `max_failures`, `timeout`, `include`/`exclude` filters, formatters, `capture_log`. `ExUnit.Filters` parses `--only`/tag expressions and file:line specs (mildly untrusted string parsing).
- **Test-author-supplied**: test/module bodies, `setup`/`on_exit` callbacks, `@tag :tmp_dir`, doctest strings. These run *arbitrary user code* inside spawned processes — expected to crash, so the runner must trap every `kind, error` catch (see `exec_test`, `exec_test_setup`, `exec_callback`).
- **Library-internal**: the `ExUnit.Server` ETS table (also used by `OnExitHandler`), the event-manager pids, monitor refs.

Implicit contracts worth watching: `Server` state machine assumes `loaded` transitions integer→`:done` monotonically (`take_sync_modules` asserts `%{waiting: nil, loaded: :done, async_groups: []}` via pattern match — a stray call there raises `MatchError`). `OnExitHandler.run/2` assumes exactly one ETS row (`[{^pid, sup, callbacks}] = :ets.take(...)`).

## Failure-mode model
- **State-threading in the module scheduler** (`Server.take_modules`/`take_until`, `restore_modules`): dropping a module from the async queue, or reversing accumulation order wrong, silently skips tests. Note the deliberate `take_until` reimplementation because `:queue.split` fails when count > size — an off-by-one here loses modules.
- **Reduce/accumulator drops** in `Runner.async_loop`, `run_module`, `run_tests` (`reduce_while` on `:max_failures_reached` — `Enum.take(invalid_tests, n)` and the `:surpassed`→`nil` path decide whether `module_finished` fires at all; the code comment flags this as fragile).
- **Missing/greedy `handle_info` / `handle_cast`**: `RunnerStats.handle_cast(_, state)` and `CaptureServer` catch-alls can *swallow* messages that should update counters — silent miscount. `CaptureServer.handle_info({:DOWN,…})` must both `remove_log_capture` and `release_device` or leak global device registration.
- **Resource/cleanup leaks**: `CaptureServer.capture_new_device`/`revert_device_to_original_pid` swap the *global* `:stdio`/named process registration; a crash between `Process.unregister` and `Process.register` corrupts global state for all subsequent tests. Same for `:logger` default-handler swap (`log_capture_on`/`maybe_revert_to_default_handler`) — the `status` must round-trip.
- **Timeout/monitor races**: `receive_test_reply`, `receive_runner_reply`, `exit_setup_all`, `terminate_supervisor` interleave `:DOWN`, kill, and reply messages; a missing `Process.demonitor(_, [:flush])` or an un-drained late message leaves stray messages that a *later* `receive` mis-matches.
- **Contract-shape violations**: `failed/3` must always return `{:failed, [{kind, reason, stack}]}`; `MultiError` special-case must not collapse the list.

## Common bug patterns for this family
- Counter arithmetic in `process_max_failures` (`previous >= max`, `previous + bump < max`, `{:reached, max - previous}`) — classic boundary off-by-one deciding `:no`/`:reached`/`:surpassed`.
- `Enum.reverse`/accumulator order flips between `add` and `take` in `Server` (built reversed, must be reversed back exactly once).
- Passing stale state (`config` vs updated `config`) through the many `CaptureServer` `put_in`/`Map.delete` device-map updates.
- Guard/typo bugs: `and`/`or` vs `&&`, `map_size(refs) == 1` (first) vs `== 0` (last) reference-counting thresholds.
- `binary_part(output, offset, output_size)` in `device_output` — offset arithmetic that can raise if `total < offset`.

## Supervisor / process structure
- `ExUnit.Server` — named `GenServer` owning the module schedule **and** the shared `OnExitHandler` ETS table (`:public`, `read`/`write_concurrency`). A crash here loses the whole suite and the ETS table.
- `ExUnit.CaptureServer` — named `GenServer` mediating global `:stdio`/named-device and `:logger` handler swaps; serialization point preventing concurrent-capture corruption. Crash leaks global registrations.
- `ExUnit.RunnerStats` — `GenServer` (a `:gen_event`-style handler) holding failure counters + failures manifest; wrong counts here mis-drive `max_failures`.
- `ExUnit.EventManager` — a `DynamicSupervisor` + `:gen_event` pair fanning events to formatters (`restart: :temporary`).
- `ExUnit.OnExitHandler.Supervisor` — `:one_for_one`, `max_restarts: 1_000_000/1s` (effectively no restart-intensity cap) protecting `start_supervised` children per test.
- `ExUnit.Runner` — not a supervisor; the driver spawning per-module and per-test `spawn_monitor` processes, trapping sigquit.

## Cross-file invariants
- **`Server` owns the ETS table that `OnExitHandler` reads/writes.** `OnExitHandler.register/add/run` assume `Server.init` already created `ExUnit.Server` as a `:named_table`; if `Server` isn't started (or crashed and restarted) those `:ets` calls raise. A per-file reviewer of `on_exit_handler.ex` cannot judge table lifetime without this.
- **`Runner` guarantees user code is caught before it reaches `RunnerStats`.** `RunnerStats` trusts that `test.state` is always one of `nil`/`{:passed|failed|invalid|skipped|excluded, _}` — enforced entirely by `Runner.failed/3` and `prepare_tests`; a new state shape would silently fall through `increment_status_counter(state, _) -> state`.
- **`Runner.run_module` promises `EM.test_started` is paired with exactly one `EM.test_finished`.** Formatters and `RunnerStats.total` counting depend on this 1:1; the `:surpassed`→`nil` early-return path is the one place that intentionally *omits* `module_finished`.
- **`OnExitHandler.run` must be invoked exactly once per registered pid** (`:ets.take` is destructive) — `Runner.exec_on_exit` is the sole caller; a double-run raises `MatchError` on the empty take.
- **`CaptureServer` assumes it is the only mutator of the named device / `:logger` default handler** during a run — reference counts (`map_size(refs)`) are only correct if all capture on/off traffic funnels through this one server.