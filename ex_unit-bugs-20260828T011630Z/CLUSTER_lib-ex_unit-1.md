# Cluster cross-file findings

#### `CSCAN-cluster-max-failures-counts-excluded-tests`

- **Finding ID** — CSCAN-cluster-max-failures-counts-excluded-tests
- **Severity** — Medium
- **Confidence** — high
- **Files involved** —
  - `runner.ex:245-266` (`run_module` — split into `to_run_tests`/`excluded_and_skipped_tests`, then `process_max_failures(config, test_module)`)
  - `runner.ex:559-561` (`process_max_failures` module clause: `bump = length(test_module.tests)`)
  - `runner.ex:571-579` (`process_max_failures/3` and its `{:reached, max - previous}` arithmetic)
  - `runner_stats.ex:28-32, 57-64` (the shared `failure_counter` that `increment_failure_counter` bumps and `get_failure_counter`/`max_failures_reached?` reads)
- **Finding shape** — State-machine / shared-counter inconsistency across files (a counter incremented in one file by a value that does not match what the reporting path in another file actually records).
- **Evidence** —
  - In `run_module`, `test_module` is `%{module.__ex_unit__() | parameters: params}` (`runner.ex:241`); its `.tests` field is the **full** compiled test list. `prepare_tests` (`runner.ex:281-305`) splits that list into `to_run` and `to_skip` (excluded/skipped) **without** mutating `test_module.tests`.
  - On a `setup_all` failure, `run_module_tests` returns `invalid_tests = Enum.map(tests, &%{&1 | state: {:invalid, test_module}})` where `tests` is `to_run_tests` only (`runner.ex:334-335`), and `test_module.tests` still holds the full list.
  - `process_max_failures(config, %ExUnit.TestModule{state: {:failed, _}, tests: tests})` then computes `bump = length(tests)` = **full** count (`runner.ex:559-561`) and calls `ExUnit.RunnerStats.increment_failure_counter(stats_pid, bump)` (`runner.ex:572`), advancing the same `failure_counter` that gates the suite (`runner_stats.ex:61-64`, `runner.ex:581-584`).
  - But only `invalid_tests` (⊆ `to_run`) are ever emitted as failures via `EM.test_started`/`test_finished` (`runner.ex:271-274`); the excluded/skipped tests were already reported separately as excluded/skipped at `runner.ex:248-251`.
  - Net effect: excluded/skipped tests in a module whose `setup_all` fails are counted toward `--max-failures`. Concrete repro: two modules, each with 1 excluded + 1 real test, both `setup_all` fail, `--max-failures=2`. Module A bumps the counter by 2 (1 excluded + 1 real), reaching the limit after only **one** real failure has been reported; module B is then suppressed as `:surpassed`. The suite aborts a full failure early.
- **What the per-file pass missed** — A `runner.ex`-only reviewer sees `length(test_module.tests)` as self-evidently "the module's tests"; the defect only appears once you cross-reference (a) that `TestModule.tests` is the *unfiltered* list per `prepare_tests`, and (b) that `RunnerStats.failure_counter` is a shared gate that must match the tests actually reported as failures — a two-module invariant.
- **Suggested fix** — In the failed-module clause, bump by the number of tests actually invalidated, i.e. pass `length(invalid_tests)` (equivalently `length(to_run_tests)`) instead of `length(test_module.tests)`. Concretely, change the clause at `runner.ex:559-561` to take the invalid-test count from the caller (thread `invalid_tests` into `process_max_failures`, or match on the module's post-run invalid list) rather than `length(tests)`.

---

Beyond this, the cluster is internally consistent on the invariants the threat model flags: `EM.test_started`↔`test_finished` pairing holds (including the intentional `:surpassed`→no-`module_finished` path), `OnExitHandler.run`'s destructive `:ets.take` is called exactly once per `register`ed pid from `runner.ex`'s `exec_on_exit`, `Runner.failed/3`'s `{:failed, [{kind,reason,stack}]}` shape matches what `cli_formatter.ex`/`formatter.ex` destructure, the `FailuresManifest` version/read/write contract is used symmetrically by `runner_stats.ex` and `filters.ex`, and the `AssertionError` default `context: :==` is always within the set `Diff.context_to_env/1` handles. The "invalidate successful tests when the module failed" branches in `runner_stats.ex:88-105` and `cli_formatter.ex:182-213` are dead under the current `runner.ex` emission (a failed module always emits only invalid tests) but are mutually consistent, so I did not report them as a defect.
