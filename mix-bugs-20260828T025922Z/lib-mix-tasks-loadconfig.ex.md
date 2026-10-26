# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/loadconfig.ex
<!-- sha256: a20aeefc6f76b93f8a4cd21851afd5b3bd44ebdb254bacdaa986b8ebc65f62b8 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:55:29Z_

## Summary

`Mix.Tasks.Loadconfig` loads and persists compile-time configuration; the only reachable defect is that its `run/1` dispatch is non-total and crashes with an uninformative `CaseClauseError` when invoked with more than one argument.

## Bugfixes

#### `CSCAN-loadconfig-run-multi-arg-caseclause`
- **Finding ID:** `CSCAN-loadconfig-run-multi-arg-caseclause`
- **Severity:** Low
- **Confidence:** high
- **Location:** `loadconfig.ex:36-39`, `run/1`
- **Class:** Missing clause / non-total function
- **Impact:** Running `mix loadconfig a b` (or any invocation with ≥2 arguments) raises a bare `CaseClauseError` with the raw argument list and no guidance. The `case args` only handles `[]` and `[file]`; every other arity falls through unmatched. The error tells the user nothing about the fact that `loadconfig` accepts at most one path, so a mistyped or glob-expanded command line produces a confusing stack trace instead of a usage message.
- **Evidence:** verified. Reproduced the exact `case` from `run/1`:
  ```
  $ elixir t.exs   # case ["a","b"] do [] -> ...; [file] -> ... end
  error: {:case_clause, ["a", "b"]}
  ```
  The `case` in `run/1` has no clause for lists longer than one element and no catch-all, so `Mix.Task.run("loadconfig", ["a","b"])` raises `CaseClauseError`.
- **Suggested fix:** Add a fallthrough clause that reports usage, e.g.
  ```elixir
  case args do
    [] -> load_default()
    [file] -> load_compile(file)
    _ -> Mix.raise("mix loadconfig expects a single config file, got: #{Enum.join(args, " ")}")
  end
  ```

## Risks / caveats

- I could not exercise the full `mix loadconfig` CLI path end-to-end inside a real project (the sandbox shell resets cwd and no Mix project is present), so reachability of the `CaseClauseError` rests on the standard `Mix.Task.run/2 → run/1` dispatch contract, where `args` is the token list after the task name; the `case` crash itself was reproduced in isolation.
- `load_default/0`'s guard `File.regular?(config_path) or config_path != "config/config.exs"` was examined and judged intentional (a user-set custom `config_path` is loaded unconditionally, and a missing custom file legitimately raises inside `Config.Reader`), so it is not reported. The other functions (`load_compile`, `load_runtime`, `persist_apps`, `preserve_config`) are `@doc false` internal helpers; their contracts were reviewed but no reachable defect via the documented public `run/1` API was found.

### Ruled out
- `load_default/0` returning `[]` vs a config list — inconsistent return type across branches, but `run/1`'s result is not a documented contract and no caller depends on it; not a behavioural bug.
- `persist_apps` reserved-app handling — it correctly filters `@reserved_apps`, emits an error via `Mix.shell().error`, and still returns `apps`; behaviour matches the moduledoc intent, verified by reading the merge/filter logic.

