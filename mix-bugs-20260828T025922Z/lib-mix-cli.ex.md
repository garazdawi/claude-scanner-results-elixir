# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/cli.ex
<!-- sha256: 33019d87162f21941309ff637745c8f16347625cf990337f58bb11cc9d15e8e3 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:09:24Z_

## Summary

`Mix.CLI` is the escript entry point that boots Mix, resolves the task and environment from CLI args/env vars, compiles `mix.exs`, and dispatches the task while translating Mix-namespaced exceptions into clean exit codes; after inventorying every clause and running the behavioural checks below, I found no genuine, in-scope defect — the file is a faithful, correct copy of upstream Mix.CLI (byte-identical to the installed `/opt/elixir` build).

## Bugfixes

No findings.

## Risks / caveats

- The only `@doc`'d public entry points are `main/2` and `preferred_cli_env/1`; everything else is private (`@moduledoc false`). I judged reachability through those two. Bugs reachable only by driving the private helpers directly are out of scope by the review rules.
- I could not diff against an independent upstream reference: the installed toolchain at `/opt/elixir` is built from the same source under review (`diff` reports the files identical), so it confirms *what runs* but is not a second opinion on *what should run*. My correctness judgement rests on reasoning about the semantics plus the targeted behavioural checks below, not on an upstream comparison.
- Atom creation via `String.to_unsafe_atom(task)` (line 167) and inside `Mix.Task.alias?/get` is real but not exploitable through `main/2`: each OS process runs `main` once, so the CLI task name cannot be varied enough within a single run to exhaust the atom table. It is also the threat-model-acknowledged intentional design. Not reported.

### Ruled out

- **`run_task/2` exception classification (lines 134–143) mis-handling non-integer `:mix` values** — I worried the guard `code in 0..255` could crash or misclassify when `exception.mix` is a non-integer. Verified: `Mix` exceptions set `mix: true` (exceptions.ex lines 6/59/69/89) or `mix: <integer>` (`Mix.Error`, line 93, set by `Mix.raise` to `exit_status`). `{false, true}` is caught by the first clause; integers by the second; everything else (including a missing `:mix` field → `Map.get` default `false`) falls to `_ -> reraise`. I confirmed `"foo" in 0..255`, `true in 0..255`, `:atom in 0..255`, `1.5 in 0..255` all return `false` without raising (`elixir -e` run), so the guard never crashes on a stray value. Classification and exit codes are correct.
- **`MIX_PROFILE` block rebinding `args` (line 30) clobbering the outer `args`** — the inner `{opts, args} = ...` shadows the argument used later by `check_for_shortcuts(args)`. Verified Elixir block scoping: assignments inside an `if` body do not leak (`elixir -e` showed the outer `args` unchanged as `[1,2,3]`). No leakage; the outer `args` reaching `check_for_shortcuts`/`proceed` is intact.
- **`preferred_cli_env/1` receiving an atom instead of a string** — `maybe_change_env_and_target` converts the task to an atom before threading it through `preferred_cli_env/3` into the public `preferred_cli_env/1`. The `/1` guard already allows `is_atom(task)`, and `Mix.Task.get/1` → `fetch/1` handles both binaries and atoms (`to_string(task)` in task.ex line 357). No `FunctionClauseError`.
- **`load_mix_exs/1` not restoring `:no_warn_undefined` if `Code.compile_file/1` raises (lines 66–69)** — a compile error in `mix.exs` leaves the global compiler option at `:all`, but the exception propagates out of `proceed` (it is not inside the `run_task` `try`) and terminates the escript, so the leaked option dies with the process. Behaviour is identical to upstream and matches the documented "compiling `mix.exs` is fatal" contract; not a defect reachable as a wrong *result*.
- **`default_task/1` / `get_task/2` receiving `nil` project** (`mix --flag` with no `mix.exs`) — `function_exported?(nil, :cli, 0)` is `false`, `Mix.Project.config()[:default_task]` is `nil`, so the `cond` falls through to `"run"`. No crash on the `nil` path.
- **`div(time, 1000)` in the `MIX_DEBUG` timing path (line 15)** — `:timer.tc/2` returns microseconds, so `div(_, 1000)` correctly yields milliseconds for the `"...ms"` message. Correct unit handling.

