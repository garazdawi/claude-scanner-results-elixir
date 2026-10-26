# Cross-file findings

## Asserted-dependency ledger

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| lib-ex_unit-callbacks.ex.md | `OnExitHandler.add` returning `:ok`/`:error` | Confirmed | on_exit_handler.ex.md §"add/3…crashing when pid absent" confirms `rescue _ -> :error`, returns `:ok`/`:error` per spec. |
| lib-ex_unit-callbacks.ex.md | `Supervisor` / `start_supervised`/`stop_supervised` delegated to stdlib, taken at documented contract | Unresolved | `Supervisor` is stdlib; no per-file report covers it, so the delegation is unverified here. |
| lib-ex_unit-capture_log.ex.md | `capture_server.ex:73–99` `:formatter`/extra-opts forwarding via `Keyword.pop`/`get_lazy` | Confirmed | capture_log.ex.md read capture_server source and executed 3 cases; matches documented contract. |
| lib-ex_unit-cli_formatter.ex.md | `repeat_until_failure` populated as integer by `ex_unit.ex:569` / `runner.ex:51` (`Keyword.fetch!`) | Confirmed | cli_formatter.ex.md verified via source; ex_unit.ex.md line 22 corroborates the same config-always-present analysis. |
| lib-ex_unit-cli_formatter.ex.md | `nil > 0` slowest/repeat comparisons safe because config keys always integers | Confirmed | ex_unit.ex.md line 22 explicitly extends its verified `slowest`/`slowest_modules` finding to "the identical comparisons in cli_formatter.ex". |
| lib-ex_unit-case.ex.md | `ExUnit.Callbacks.__describe__/3,4` logic (describe/2 delegation) | Unresolved | callbacks.ex.md does not analyze `__describe__`; only case.ex's local AST wiring was checked. |
| lib-ex_unit-case.ex.md | internal `:ex_unit`/`:breakpoints` env (annotate_test) not public API | Unresolved | No report covers this internal env; treated as out-of-scope misuse, unverified. |
| lib-ex_unit-diff.ex.md | `ExUnit.Assertions` only ever produces valid AST for Diff | Unresolved | assertions.ex.md finds a bad-AST path (`mark_as_generated`) but does not address the AST handed to `Diff`; dependency unverified. |
| lib-ex_unit-runner_stats.ex.md | `Runner.process_max_failures/3` (runner.ex:571-578) uses pre-increment `increment_failure_counter` return deliberately | Confirmed | runner.ex.md line 57-58 verifies process_max_failures boundary arithmetic against that binding. |
| lib-ex_unit-runner_stats.ex.md | `test.state` shapes bounded by `Runner.failed/3`/`prepare_tests` (increment_status_counter fall-through) | Unresolved | runner.ex.md does not enumerate the exhaustive set of state shapes produced; invariant is asserted (also in THREAT_MODEL), not verified in the caller report. |
| lib-ex_unit-server.ex.md | `OnExitHandler` destructive `:ets.take` single-row contract | Confirmed | on_exit_handler.ex.md §"run/2…take raising on missing row" documents the one-register/one-run destructive-take contract. |
| lib-ex_unit-on_exit_handler.ex.md | `Runner.exec_on_exit` (runner.ex:541) invokes `run/2` exactly once per pid | Unresolved | runner.ex.md does not verify the exactly-once-per-pid caller invariant that the `:ets.take` match relies on. |
| lib-ex_unit-event_manager.ex.md | internal cast API (`notify`/`stop`/`*_started`) reachable only via `Runner` internals | Unresolved | Undocumented internal surface; no caller report substantiates unreachability of the `stop/1` `:noproc` race. |
| lib-ex_unit-capture_server.ex.md | `log/2` `:logger` handler callback exercised only indirectly | Unresolved | Driven by `:logger`, not the public API; no report exercises the drain/trap-exit path. |
| lib-ex_unit-mix.exs.md | default env values (`assert_receive_timeout`, `max_failures`, `rand_algorithm`, `timeout`) correct only vs consumers in ex_unit.ex/runner.ex/assertions.ex/server.ex | Unresolved | mix.exs.md explicitly did not adjudicate consumers; consumer reports do not confirm these specific defaults are mishandled or well-handled. |

## Findings

No cross-file findings.

Every asserted cross-file dependency resolved to either **Confirmed** (the cited file's report substantiates the contract) or **Unresolved** (nobody checked the invariant). No report contradicts another's claim, and no "safe" verdict rests on an inherited risk that a caller's report refutes. The Unresolved rows are coverage gaps recorded for a human/later pass — per the ledger protocol they are not promoted to findings.
