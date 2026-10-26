# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/cli_formatter.ex
<!-- sha256: 7bf48a0d9e4df4aef730e95b1b86494a20d05bc34b429318e0a0abd722a9e21a -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:30:34Z_

## Summary

The default ExUnit console formatter is largely correct, but its summary-line coloring uses a broken condition that paints a fully-passing run yellow whenever the number of passing tests happens to equal the number of excluded tests.

## Bugfixes

#### `CSCAN-cli_formatter-excluded-eq-passed-miscolor`

- **Finding ID:** `CSCAN-cli_formatter-excluded-eq-passed-miscolor`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/ex_unit/lib/ex_unit/cli_formatter.ex:415`, `print_summary/2` (the third `cond` clause)
- **Class:** Operator / precedence (wrong condition — stale assumption about `test_counter` contents)
- **Impact:** A test run with **no** failures and **no** invalid tests, where the count of passing tests equals the count of excluded tests, prints its "Result:" summary in the yellow *invalid* color instead of the green *success* color. The text is correct; only the color misleads, suggesting something went wrong when the suite passed cleanly.
- **Evidence:** `verified`.

  The clause is `test_counter > 0 && config.excluded_counter == test_counter`. Here `test_counter` comes from `collect_test_counter/1`, which sums only the `test_counter` map — and the excluded-test handler (`cli_formatter.ex:94-99`) increments **only** `excluded_counter`, never `test_counter`. So `test_counter` holds the passed(+failed) count, and this clause fires exactly when `passed_count == excluded_count`, not "all tests excluded". (The genuine all-excluded case has `test_counter == 0`, so this clause — guarded by `test_counter > 0` — can never serve it; that case is handled separately at line 59 and falls through to the green `success` branch.)

  Driving it through the public API (`ExUnit.start`/`ExUnit.run` with an `exclude` filter), forcing colors on:

  ```
  # 1 passing test, 1 excluded test:
  ^[[33m                       <- yellow (invalid)
  Result: 1 passed, 1 excluded^[[0m

  # 2 passing tests, 1 excluded test:
  ^[[32m                       <- green (success)
  Result: 2 passed, 1 excluded^[[0m
  ```

  The only difference between the two runs is the passing count; when it equals the excluded count the summary flips to yellow. `1p1e` is colored `[33m` (yellow), `2p1e` is `[32m` (green).
- **Suggested fix:** The clause is dead for its apparent purpose and misfires on a coincidence. Either delete it (letting fully-passing runs always hit the green `true -> success` branch), or, if a yellow "everything was excluded" summary is desired, test the real condition — e.g. `test_counter == 0 && config.excluded_counter > 0` — consistent with the check already used at line 59.

## Risks / caveats

- The mis-coloring is cosmetic: the summary text (`Result: N passed, M excluded`) is always numerically correct; only the ANSI color of that line is wrong, and only when colors are enabled. Hence Low severity.
- `CLIFormatter` is `@moduledoc false`, but it is the default formatter installed by `ExUnit.start/1`, so this path is reached by ordinary documented usage (running a suite with `exclude` filters). I treated the formatter's cast/event interface driven by the runner as the effective public surface.
- I ran suites against `elixir`/`ExUnit` built from this source (reported `ex_unit 1.21.0-dev`) to confirm behavior; I could not diff against a pinned upstream tag (no VCS metadata in the tree), so the "intended" reading of line 415 is inferred from the surrounding code, not from upstream history.

### Ruled out

- **`init/1:16` `opts[:repeat_until_failure] > 0` with a `nil` value** — Erlang term ordering makes `nil > 0` true, which would wrongly print the `remaining_runs` suffix. Ruled out (verified via source): the key is populated by `ExUnit`'s `persist_defaults`/`configuration` and read elsewhere with `Keyword.fetch!` (`ex_unit.ex:569`, `runner.ex:51`), so it is always an integer when the formatter sees it; a caller producing `nil` would be violating the internal config contract, out of scope.
- **`format_slowest_tests/format_slowest_modules:306,335` division by `run_us`** — `slowest_us / run_us` would raise `ArithmeticError` if `run_us == 0`. Ruled out (verified): `times_us.run` is wall-clock microseconds and is non-zero even when every test is skipped/excluded — the `slowest: 2` + all-skipped run printed `0.0% of total time` without crashing, so the zero-divisor case is not reachable through the normal suite lifecycle.
- **`update_test_counter/292` `FunctionClauseError` when a test lacks a `:test_type` tag** — Requires a `%ExUnit.Test{}` whose `tags` map omits `:test_type`; ExUnit always injects `test_type`, so reaching this needs a hand-crafted struct violating the internal contract. Out of scope.
- **`:excluded`/`:skipped` handlers guarded by `is_binary(reason)` (lines 95, 102)** — A non-binary reason falls through to the catch-all `handle_cast(_, config)` and is silently uncounted, but ExUnit always supplies a binary reason string; reaching the fallthrough requires an internally-malformed event. Out of scope.
- **`module_finished {:failed}` reduce (lines 189-199) double-counting** — Verified via a `setup_all`-raising module: the invalidated tests carry `{:invalid, …}` state (not `nil`), so the reduce correctly skips them and no double count occurs; the `nil`-state branch is reserved for module-level teardown failures after tests already passed, where re-counting them as failures is intended.

