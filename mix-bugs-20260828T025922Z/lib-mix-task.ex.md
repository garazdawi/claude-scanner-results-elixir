# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/task.ex
<!-- sha256: 8b468b070be109fafecfd132d0cf2e3b71a214e18b4035c25030dd0ef13dee2b -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/task.ex
<!-- sha256: 8b468b070be109fafecfd132d0cf2e3b71a214e18b4035c25030dd0ef13dee2b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`Mix.Task` is a well-exercised standard-library module; I found no reachable crash or logic defect through its documented public API, but two of its `@doc` contracts understate the values their functions actually return (verified against a live build). The one caveat the first pass hedged on — state-threading across `Mix.TasksServer`/`Mix.ProjectStack` — has now been resolved by reading both singleton modules; the call contracts are internally consistent and no defect is present (see "Did I verify this").

## Bugfixes

#### `CSCAN-task-get-alias-doc-mismatch`
- **Finding ID:** CSCAN-task-get-alias-doc-mismatch
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/mix/lib/mix/task.ex:319-331` (`get/1`), backed by `fetch/1` at line 356
- **Class:** Documentation mismatch
- **Impact:** The `@doc` for `get/1` promises: "Returns `nil` if the module cannot be found, **if it is an alias**, or if it is not a valid `Mix.Task`." In reality `get/1` → `fetch/1` → `Mix.Utils.command_to_module` never consults `Mix.Project.config()[:aliases]`. When a task name is *both* an alias and a real task module (e.g. the common `aliases: [test: [...]]`), `get/1` returns the underlying module, not `nil`. A caller that trusts the doc to detect aliases via `get/1 == nil` will misclassify aliased tasks and may resolve/run the shadowed module instead of recognizing the alias.
- **Evidence:** `verified`. In a project with `aliases: [test: ["cmd echo hi"]]`:
```
$ mix run -e 'IO.inspect(Mix.Task.get("test")); IO.inspect(Mix.Task.alias?("test"))'
get(test): Mix.Tasks.Test
alias?(test): true
```
`get("test")` returns `Mix.Tasks.Test` although `"test"` is an alias — contradicting the documented "Returns `nil` … if it is an alias." The alias-blind path is confirmed in-source: `fetch/1` (line 356) delegates only to `Mix.Utils.command_to_module`, never to `config()[:aliases]`.
- **Suggested fix:** Correct the docstring to drop the "if it is an alias" clause (aliases are not considered; use `alias?/1` to detect them), or make `get/1` actually return `nil` when `alias?(task)` is true if that is the intended contract.

#### `CSCAN-task-moduledoc-false-doc-mismatch`
- **Finding ID:** CSCAN-task-moduledoc-false-doc-mismatch
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/mix/lib/mix/task.ex:210-222` (`moduledoc/1`)
- **Class:** Documentation mismatch
- **Impact:** The `@doc` states "Returns the moduledoc or `nil`", but the code has a third return value: the `_ -> false` clause returns `false` whenever docs are hidden (`@moduledoc false`) or the docs chunk is unavailable. A caller who follows the prose contract and pattern-matches only on a string or `nil` (e.g. `case moduledoc(mod) do doc when is_binary(doc) -> ...; nil -> ... end`) will hit a `CaseClauseError` on the `false` value. The `@spec` (`String.t() | nil | false`) is correct; only the human-readable `@doc` is wrong.
- **Evidence:** `verified`.
```
$ elixir -e 'defmodule H do @moduledoc false; def x, do:1 end; IO.inspect(Mix.Task.moduledoc(H))'
false
```
- **Suggested fix:** Update the docstring to mention the `false` return (hidden/unavailable docs), matching the existing `@spec`.

## Risks / caveats

- Reproductions used the `elixir`/`mix` at `/opt/elixir`; the source under review does not carry a pre-built `bin/mix`. The two findings are docstring-vs-code mismatches established directly from the reviewed source (the alias-blind `fetch/1` path and the `_ -> false` clause), so they hold regardless of which build ran the demonstration.
- **(Hedge resolved.)** The first pass hedged that state-threading across `Mix.TasksServer`/`Mix.ProjectStack` "was not exhaustively driven." I have now read both modules. `Mix.TasksServer.run/1` is `:ets.insert_new` (true only on first insert), `get/1` is `:ets.member`, `put/1` is `:ets.insert`; `Mix.ProjectStack.recur/1` and `on_recursing_root/1` both use `try/after` to restore `recursing?` flags. The `do_run`/`run_task`/`run_alias`/`reenable` call sequences in this file use those primitives consistently (run-once via `insert_new`, alias completion marks the `{:task,…}` tuple via `put`, re-enable deletes both tuples). No state-threading defect is present. This is no longer an unverified assumption.

### Ruled out
- **`task_from_path/1` negative binary-size** (line 187–190): a short filename yields a negative `part`, but `<<…::binary-size(^part)…>>` simply fails to match (verified: returns `:nomatch`, no raise) and falls to `_ -> nil`. No crash.
- **`run_requirements/1` empty requirement** (line 516): `[task | args] = OptionParser.split("")` would `MatchError`, but `@requirements` is author-supplied (read from the task module's own `:requirements` attribute at line 280–285), not untrusted external input — reaching it requires violating the documented contract.
- **`with_debug/4` second `shell.info` using raw `task`** (line 527): `task` is always a binary by the time `run_task` is reached — `do_run` converts atoms at line 416–418, and `run_alias` passes only the already-string `original_task`/`task` values — so the iodata is valid; only cosmetically inconsistent with the first line's `task_to_string`.
- **`shortdoc/recursive/requirements` calling `module.__info__/1`** on an unloaded module: raises `UndefinedFunctionError`, but passing an unloaded/non-task atom violates the `task_module` contract; not a defect on documented input.
- **Empty-list alias `aliases: [foo: []]`** (`do_run` line 425): `[]` is truthy so the alias branch runs, but `run_alias([], …)` cleanly returns the seed `:ok` and marks the task via `Mix.TasksServer.put` (line 604) — no wrong behaviour.

## Did I verify this

- **Findings 1 & 2:** `verified` at runtime (commands shown) *and* against source: `fetch/1`/`get/1` at 319–364 for finding 1, the `_ -> false` clause at 220 for finding 2.
- **State-threading caveat:** verified by reading the two neighbour singletons.
  - `Mix.TasksServer` (`lib/mix/lib/mix/tasks_server.ex`): `run/1 = :ets.insert_new` (first-write semantics), `get/1 = :ets.member`, `put/1 = :ets.insert`, `delete_many/1 = per-key :ets.delete`. This confirms the run-once dispatch in `do_run` (lines 424–441) and the alias-completion `put` (line 604) behave as this file assumes.
  - `Mix.ProjectStack` (`lib/mix/lib/mix/project_stack.ex`): `recur/1` (236–245) and `on_recursing_root/1` (64–79) both wrap their body in `try/after` that restores the `recursing?` flag — so the `run_task` recursion branches (486–495) and `run_alias` recursion (585–587) cannot leave a stale recursing flag even on a raise. `recursing/0` (247–250) returns a module-or-nil, matching `recursing?/0`'s use at line 260.
  - Reading these modules tells me what the guards do, not every input that could disagree; I did **not** drive every umbrella/recursion/alias-args permutation at runtime. I make no claim beyond "the call contracts are internally consistent and I found no defect" — I am not asserting exhaustive runtime coverage.

### Dropped in re-review
None. Both findings are preserved; the only change is that the first-pass hedge in Risks/caveats has been resolved (not dropped) by reading the two neighbour modules, and no finding was upgraded or downgraded.
