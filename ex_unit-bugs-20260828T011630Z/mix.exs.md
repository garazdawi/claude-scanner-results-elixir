# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/mix.exs
<!-- sha256: d7383385a3958aaeaceb809cb52bcfc74ef8485611045130bb2b6608dd246ef4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:37:50Z_

## Summary

`mix.exs` is ExUnit's build/application-metadata declaration (project config, registered process names, and default application env); it contains no runtime logic reachable through a documented public API, and I found no defects.

## Bugfixes

No findings

This file is a static `Mix.Project` module. `project/0` and `application/0` are the two documented Mix callbacks, and both return plain literal keyword lists — no computation, no branching, no state, no message handling. The only content of substance is the `env:` list of default `ExUnit` configuration values, which is consumed elsewhere (`ExUnit.configure/1`, `ExUnit.Server`, `ExUnit.Runner`), not here. There is nothing in this file whose behavior can be wrong on any input, because it takes no input.

## Risks / caveats

- The correctness of the default env values (e.g. `assert_receive_timeout: 100`, `max_failures: :infinity`, `rand_algorithm: :exsss`, `timeout: 60_000`) can only be judged against their *consumers*, which live in other files (`ex_unit.ex`, `runner.ex`, `assertions.ex`, `server.ex`). A wrong default here would surface as a bug in those files' logic; this review is scoped to `mix.exs` alone, so I did not adjudicate whether any consumer mishandles these values.
- `registered:` lists four names (`ExUnit.CaptureServer`, `ExUnit.OnExitHandler`, `ExUnit.Server`, `ExUnit.Supervisor`); whether every one is actually registered at startup (and none missing) is a property of the supervision tree in `ex_unit.ex`/`server.ex`, not verifiable from this file.

### Ruled out

- **`version: System.version()`** — mirrors Elixir's own version for a bundled stdlib app; this is the intended convention for in-tree apps, not a defect.
- **Commented-out `max_cases`/`seed` env keys** — the comment states they are "Calculated on demand," matching ExUnit's runtime computation of `System.schedulers_online() * 2` and a random seed; their absence from the literal list is deliberate, not a missing default.
- **No `handle_*`/`receive`/resource-allocation code** — I read the entire 47-line file; it declares no processes, opens no resources, and defines no callbacks beyond the two pure Mix project callbacks, so the process/leak/message-handling bug classes have no surface here.

