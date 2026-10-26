# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/examples/one_of_each.exs
<!-- sha256: ebcc589f78d4b2e98944531c5724f332413b2a527d89011cbbb332b66afd2ec0 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:19:58Z_

## Summary

`one_of_each.exs` is a runnable demonstration script whose entire, documented purpose is to produce "one of each type of failing test," so its deliberately-failing tests are the intended behavior, not defects — no bug reachable through any public API exists here.

## Bugfixes

No findings

## Risks / caveats

- This file is an **example/demonstration script** (`lib/ex_unit/examples/one_of_each.exs`), not a library module. Its own `@moduledoc` states: *"This module contains one of each type of failing test. It is used simply to document the style of each."* Every test (`assert [@one] = [@two]`, `flunk("we failed. totally")`, `1 / val` in `ignite/0`, the `assert_receive` timeout, etc.) is **intentionally** written to fail so the ExUnit formatters can be inspected. Failure is the contract, so none of these are bugs.
- The file exposes **no documented public API**: `TestOneOfEach` is a test module invoked only by `ExUnit.start/run`; `some_vars/2`, `blows_up/0`, and `ignite/1` are `defp` (private) and reachable only from within the module's own tests. Under the scope rules (bugs must be reachable through documented public entry points, and intentional failures/private helpers are excluded), there is nothing in scope to report.
- I read the whole file. The only "surprising" constructs — `spawn_link` with an empty `receive do end` (test 25), `raise ExUnit.MultiError` (test 26), and the trap-exit tasks (tests 32–33) — are each the *point* of the corresponding example (illustrating exit errors, multi-errors, and trapped linked crashes), and they behave exactly as documented.

### Ruled out

- **`ignite/1` division by zero (`1 / val` with `val = 0`, test 24)** — not a defect: `blows_up/0` exists specifically to raise `ArithmeticError` mid-test so test 24 ("exception raised while running test") demonstrates that failure style. The failure is the documented intent of the example.
- **`assert_receive :no_message_after_timeout` (test 12) hanging/failing** — intended; it demonstrates the receive-timeout failure formatting, and `assert_receive`'s default 100 ms timeout bounds it, so it does not hang.
- **`@long_data_1 = @long_data_2` match failures (tests 3, 7, etc.)** — intended failing/never-failing matches used to show long-argument diff rendering; behavior matches the module doc.

