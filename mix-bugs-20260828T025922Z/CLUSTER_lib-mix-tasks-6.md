# Cluster cross-file findings

#### `CSCAN-cluster-coverage-exit-status-overrides-test-failure`

- **Finding ID** — CSCAN-cluster-coverage-exit-status-overrides-test-failure
- **Severity** — Medium
- **Confidence** — high
- **Files involved** —
  - `lib/mix/tasks/test.ex:663-664, 718, 725-736, 1084-1086` (chooses the test-failure exit status and registers it via `System.at_exit`)
  - `lib/mix/tasks/test.coverage.ex:311-322` (registers a competing `System.at_exit` exit code 3 for a below-threshold coverage summary)
  - `/opt/elixir/lib/elixir/lib/kernel/cli.ex:134-166` (out-of-cluster: the `at_exit` fold that resolves the winner)
- **Finding shape** — State-machine inconsistency / inherited-assumption broken (two files register uncoordinated exit-status hooks; resolution order is an accident of registration order).
- **Evidence** —
  - `test.ex:718` runs the coverage callback first: `cover && cover.()`. That callback is the closure returned by `Mix.Tasks.Test.Coverage.start/2`, which calls `generate_cover_results/…` → `summary/3`.
  - `test.coverage.ex:316-319`: when `totals < get_threshold(...)`, it does `System.at_exit(fn _ -> exit({:shutdown, 3}) end)` — registered **first**.
  - `test.ex:733-736`: only afterwards, in the `cond`, for `failures > 0` it registers `System.at_exit(fn _ -> exit({:shutdown, exit_status}) end)` where `exit_status = Keyword.fetch!(ex_unit_opts, :exit_status)` (default 2, or the user's `--exit-status`) — registered **second**.
  - `cli.ex:135` reads hooks as a LIFO list (`[fun | &1]` at registration); `cli.ex:136` `Enum.reduce(hooks, res, &exec_fun/2)` processes the most-recent hook first and the **first-registered hook last**, so the coverage hook's `{:shutdown, 3}` is the final accumulator and wins. A failing suite with `--cover` and below-threshold coverage therefore exits **3**, not the `exit_status` (2, or a user-chosen `--exit-status N`) that `test.ex` deliberately selected for test failures.
- **What the per-file pass missed** — A `test.coverage.ex` reviewer sees only its own `exit 3`; a `test.ex` reviewer sees only its own `exit exit_status` hook and an opaque `cover.()` call. Only reading both files plus the `at_exit` fold order reveals that the coverage hook silently overrides the test-failure exit code.
- **Suggested fix** — Coordinate the two exit paths: have `test.ex` own the final exit status (e.g. pass the failure/exit-status decision into the coverage summary, or have `test.coverage.ex`'s summary return the threshold result to `test.ex` rather than registering its own `System.at_exit`) so a real test failure's `exit_status` takes precedence over the coverage-threshold code.

#### `CSCAN-cluster-coverage-threshold-exit-doc-mismatch`

- **Finding ID** — CSCAN-cluster-coverage-threshold-exit-doc-mismatch
- **Severity** — Low
- **Confidence** — high
- **Files involved** —
  - `lib/mix/tasks/test.ex:377-383` (moduledoc for the `:summary` / `:test_coverage` option)
  - `lib/mix/tasks/test.coverage.ex:316-319` (the actual threshold implementation)
- **Finding shape** — Other: documented-contract vs implementation mismatch across files.
- **Evidence** —
  - `test.ex:380`: “The task will **exit with status of 1** if the total coverage is below the threshold.”
  - `test.coverage.ex:318`: the code that implements exactly that threshold check does `exit({:shutdown, 3})`, i.e. exit status **3**, not 1.
- **What the per-file pass missed** — The behaviour is documented in `test.ex` but implemented in `test.coverage.ex`; neither single-file reviewer sees both the promise and the code that breaks it.
- **Suggested fix** — Update the `:summary` documentation in `test.ex:380` to state the actual exit status (3), or change `test.coverage.ex:318` to the documented value; keep the two in sync.
