# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit.ex
<!-- sha256: 8c3b3e6b889c25d2a342d7688b5f8f2711b056deeb4eceda73d4a441c3d129a3 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:21:21Z_

## Summary

`ex_unit.ex` is the mature top-level ExUnit module (application callback plus the documented `start`/`run`/`configure`/`configuration`/`plural_rule`/`after_suite`/`async_run`/`await_run`/`fetch_test_supervisor` API); I found no defect reachable through its documented public API.

## Bugfixes

No findings

## Risks / caveats

- I reviewed only `ex_unit.ex`. Its behavior depends heavily on collaborators (`ExUnit.Server`, `ExUnit.Runner`, `ExUnit.CaptureServer`, `ExUnit.OnExitHandler`, the CLI formatter) and on the application-env defaults declared in `mix.exs`; several claims of safety below rest on those defaults, which are out of this file but were checked directly.
- The `System.at_exit`/autorun path (lines 285–304) and the repeat-until-failure loop (`maybe_repeated_run/4`) can only be exercised through a full suite run driving real modules; I reasoned about them and cross-read the `Runner.run/2` return contract `{stats, {async_modules, sync_modules}}`, but did not execute an end-to-end repeated-failure suite.

### Ruled out

- **`put_slowest/1` uses `opts[:slowest] > 0` / `opts[:slowest_modules] > 0` (lines 612–618), and `nil > 0` is `true` in Elixir term ordering** — I verified `nil > 0 == true` (`elixir -e 'IO.inspect(nil > 0)'` → `true`), so if either key were absent, `put_slowest` would wrongly force `trace: true` (and thereby `max_cases: 1`). It is not reachable: `mix.exs` `application/0` sets `env: [slowest: 0, slowest_modules: 0, ...]`, and I confirmed after `Application.load(:ex_unit)` that `Application.get_env(:ex_unit, :slowest) == 0` and `:slowest_modules == 0`. `configuration/0` reads via `Application.get_all_env(:ex_unit)`, so the keys are always present as integers whenever the (loaded) app is in use; supplying `nil` requires `configure(slowest: nil)`, which violates the `non_neg_integer()` spec (out-of-scope misuse). Same reasoning covers the identical comparisons in `cli_formatter.ex`.
- **`max_cases/1` `cond` with `max = opts[:max_cases] -> max` (lines 620–626)** — `max_cases` defaults to being computed (`System.schedulers_online() * 2`) and the key is absent by default (commented out in `mix.exs`), so `opts[:max_cases]` is `nil`, the clause is falsy, and the `true ->` branch runs. A configured `0` would be truthy and returned, but `max_cases` is specced `pos_integer()`, so that is caller misuse, not a defect.
- **`after_suite/1` prepends to the callback list (lines 531–534) vs. the "reverse order / last set is first called" doc** — prepending makes the last-registered callback the list head; the runner invokes the list head-first, so documentation and implementation agree.
- **`start/1` returning `:ok` under `@spec ... :: :ok` (lines 280–305)** — both branches (`System.at_exit/1` result and the `else :ok`) yield `:ok`; `System.at_exit/1` returns `:ok`.
- **Nested `System.at_exit` registration inside the autorun exit callback (line 296)** — registering an at-exit handler from within an at-exit handler is supported and is the intended mechanism for setting the failure exit status; not a defect.
- **`run/1` raising `ArgumentError` for a non-ExUnit module (lines 478–484)** — raising on invalid input here is the documented/expected contract, and the message names the offending module, so it is actionable.

