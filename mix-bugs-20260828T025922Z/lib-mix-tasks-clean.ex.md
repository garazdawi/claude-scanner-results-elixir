# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/clean.ex
<!-- sha256: c6a82c163abcf14c6ff5796acd2eca105fd3cda1f5738900bbcc98f9882f52a8 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:37:55Z_

## Summary

`clean.ex` is the `mix clean` task — it deletes build artifacts for the current project (and optionally deps), scoped by environment. The logic is faithful to Mix's standard implementation; I found no reachable behavioural defects.

## Bugfixes

No findings.

## Risks / caveats

- The review rests on the documented public entry point `run/1` (the `Mix.Task` `run/1` contract). The internal helpers (`do_run/1`, `loadpaths!/0`) are only reachable through `run/1`, and I traced every branch reachable from a well-behaved CLI invocation (`--deps`, `--only`, neither, both).
- I verified (`verified`) that the string/glob construction is total for all option shapes a caller can produce: with `--only` absent, `opts[:only]` is `nil` and `"*#{opts[:only]}"` collapses to `"*"` (matches all env dirs, the documented default of "works across all environments"); with `--only prod` it becomes `"*prod"`. Confirmed via `elixir -e` (`"*#{nil}" == "*"`, `Path.join("_build", "*#{nil}") == "_build/*"`).
- I verified `function_exported?(nil, :clean, 0)` returns `false` (not a raise), so the comprehension filter is safe when `Mix.Task.get("compile.#{compiler}")` returns `nil` for an unknown compiler task — no `FunctionClauseError`/`ArgumentError` there.
- I did not exercise the full runtime against a live umbrella/project (the file depends on `Mix.Project`, `Mix.Task.Compiler`, `Mix.Compilers.Protocol` state that requires a booted Mix project), so the ruled-out items below are reasoned from the standard semantics of those helpers rather than an end-to-end `mix clean` run.

### Ruled out

- **`--only` glob over-matching (`"*prod"` matching more than the `prod` env dir)** — Reachable in principle: `--only dev` yields glob `_build/*dev`, which would also match a hypothetical `_build/xdev`. But this is Mix's long-standing intended construction: the leading `*` is required so the no-`--only` default (`"*"`) matches *all* environments, per the documented "works across all environments" behaviour; without the `*`, the default would collapse to `Path.join("_build", "")` and rm_rf the entire `_build` tree. The prefix over-match is a documented-behaviour quirk, not an introduced defect, and environment names sharing a suffix is not a realistic configuration. Verified the string construction with `elixir -e`.
- **`app` being `nil` in an umbrella root (`else` branch, `"lib/#{app}"`)** — At an umbrella root `Mix.Project.config()[:app]` is `nil`, producing glob `_build/*/lib/` which matches nothing meaningful. This is the standard Mix behaviour: `@recursive true` drives the real cleaning into the child apps where `:app` is set, so the root invocation cleaning nothing is correct, not a bug.
- **`Mix.Task.get/1` returning `nil` in the compiler comprehension** — `function_exported?(nil, :clean, 0)` returns `false` (verified), so unknown compiler tasks are filtered out rather than crashing.
- **Resource/lock leak** — Cleaning runs inside `Mix.Project.with_build_lock/1`, whose release is that helper's responsibility (out of this file); `run/1` itself acquires no file/port/ETS resource that could leak on an error path.

