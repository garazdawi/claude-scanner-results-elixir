# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/examples/difference.exs
<!-- sha256: 8d3f4595f22e51d2c0664a62610630f7c8ee5c94ad939e93b6cd197867e4dac4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:19:38Z_

## Summary

The file is an ExUnit example script of intentionally-failing tests that demonstrate diff highlighting; it exposes no documented public API and contains no reachable defect.

## Bugfixes

No findings

This file (`difference.exs`) is a runnable demonstration script, not library code. Its entire purpose, stated in the module `@moduledoc` ("This module contains failing tests to see difference highlighting in action"), is for every `assert` to fail so a human can eyeball ExUnit's `ExUnit.Diff` output. It defines no functions callable by any other code, exports no public API, and is not compiled into the ExUnit library — it is run directly via `bin/elixir lib/ex_unit/examples/difference.exs`. There is no caller for whom a "wrong" behavior could cost anything; the failures *are* the intended output.

## Risks / caveats

- **Scope:** The review target is a demo `.exs` with no public API surface, so the entire finding class ("bugs reachable through documented public APIs") is vacuous here — there is nothing an external caller can invoke. Any real defect would live in the modules this script exercises (`ExUnit.Assertions`, `ExUnit.Diff`, `ExUnit.Formatter`), none of which are in this file.

### Ruled out

- **All 30+ `assert`s "fail"** — Not defects. Each is a deliberately-false comparison (`491_512_235 == 490_512_035`, `42.0 == 43.0`, `assert ^x = 2`, `assert_received x when x == :hello` with an empty/mismatched mailbox, etc.) whose failure is the documented purpose of the file per its `@moduledoc`. Verified by reading: every assertion's operands are mismatched by construction and the module header states the tests are meant to fail.
- **`defmacrop one, do: 1` used in `assert one() = 2` and `assert_received one()`** — Correct usage. The private macro expands to the literal `1`; `1 = 2` and matching `1` against the sent message `12` both fail as intended, exercising the diff path for macro-expanded patterns. No hygiene or quoting defect: the macro takes no arguments and returns a literal.
- **`make_ref()`/`self()` in list/tuple tests producing non-deterministic values** — Intended. These feed runtime-unique terms into the diff renderer precisely to show how refs/pids are diffed; non-determinism of the printed value is expected for a visual demo and affects no assertion outcome (all still fail).

