# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/exceptions.ex
<!-- sha256: 7476faa6241c7e339f0491169060f3f1ba4b4ac267b33ecda162951129b86afc -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:16:41Z_

## Summary

`exceptions.ex` defines Mix's exception structs (`Mix.NoTaskError`, `Mix.InvalidTaskError`, `Mix.ElixirVersionError`, `Mix.NoProjectError`, `Mix.Error`) plus the "did you mean" task-suggestion helper; the code is small, total over its documented inputs, and I found no reachable defect.

## Bugfixes

No findings

## Risks / caveats

- The one input that crashes `Mix.NoTaskError.exception/1` is `task: nil` (or a non-string task), because `String.jaro_distance(source, nil)` raises `FunctionClauseError` inside `did_you_mean/1`. This is out of scope: the documented usage is `raise Mix.NoTaskError, task: <string>` (as done in `Mix.CLI`), so a `nil` task violates the contract. Not reported.
- I did not execute the full `did_you_mean/1` path end-to-end (it requires a started Mix instance and `Mix.Task.load_all/0`); I verified the arithmetic-sensitive primitive (`String.jaro_distance`) directly and reasoned about the reduce/threshold logic, which is total for string inputs.

### Ruled out

- **`Mix.Error` uses `mix: 1` instead of `mix: true`** — not a bug. `Mix.CLI.run_task/2` (cli.ex:134-139) reads this field as `{false, code} when code in 0..255 -> simplified_exception(exception, code)`, i.e. the integer is deliberately the process exit code. `mix: 1` is truthy and yields exit status 1, matching the intent; the boolean `mix: true` used by the other exceptions is handled by the separate `{false, true}` clause (also exit 1). Verified by reading the consuming `case` in cli.ex.
- **`max_similar/3` tie-handling / empty accumulator** — `Enum.reduce(..., {nil, nil, 0}, ...)` with `if score < current, do: best, else: {mod, source, score}`. `String.jaro_distance` returns a float in `[0.0, 1.0]` (verified: `1.0` for equal, `0.0` for disjoint/empty), so every candidate has `score >= 0` and the seed is always replaced when the module list is non-empty; on an empty list the seed `{nil, nil, 0}` flows to the `_otherwise` clause of `msg/1` (score `0` fails the `> 0.8` guard) and produces a valid message. No crash, no wrong branch.
- **`msg/1` first clause `{mod, ^task, _score}` is dead / omits `maybe_no_project()`** — not dead: it is reachable when a module's `Mix.Task.task_name/1` equals the requested task but `Mix.Task.get/1` failed because the module is not named `Mix.Tasks.*` (exactly the "rename it" message's scenario). Omitting the no-project note in that branch is intentional wording, not a defect.
- **Other exception `exception/1` builders** (`Mix.InvalidTaskError`, `Mix.ElixirVersionError`) and the struct-only `Mix.NoProjectError` — pure string interpolation over the provided opts with no branching or partial functions; nothing to fail on documented inputs.

