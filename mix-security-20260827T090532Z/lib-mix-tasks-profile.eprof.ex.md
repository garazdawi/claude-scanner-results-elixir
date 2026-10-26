# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.eprof.ex
<!-- sha256: 0249292b4c9d60f82d66bbf755f9ed6ef10e9d2ed37b16c5b730148ccccf2fe3 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.eprof.ex
<!-- sha256: 0249292b4c9d60f82d66bbf755f9ed6ef10e9d2ed37b16c5b730148ccccf2fe3 -->

_Re-reviewed on 2026-08-27_

## Summary

`Mix.Tasks.Profile.Eprof` is a developer-invoked profiling task whose only code-execution and atom-creation sinks are driven by operator-supplied command-line arguments (the `-e`/`--eval` expression, the positional script-file path, `--require`/`-r` glob patterns, and `--matching`). Having now read the delegated data-flow in `Mix.Tasks.Run.run/5`, I confirm that no dependency-, archive-, network-, or project-file-derived data crosses a trust boundary into any sink in this file or in the `run.ex` code it drives.

## Security

No findings.

### Ruled out

- **`Code.string_to_quoted!` + `Code.compile_quoted` in `profile_code/2` (lines 163, 170) — arbitrary code execution.**
  This is the documented purpose of the task: `mix profile.eprof -e "<code>"` and `mix profile.eprof my_script.exs` run exactly the expression/script the operator typed. The two callbacks passed to `Mix.Tasks.Run.run/5` are `&profile_code(&1, opts)` (fed the `--eval` string at `run.ex:214`) and `&profile_code(File.read!(&1), opts)` (fed the positional `file` at `run.ex:167`, whose value is `head`'s first element — operator-typed argv, `run.ex:135-141`). `profile.eprof` is a standalone developer command, never part of `deps.get`/`compile`/`archive.install`, so no dependency/archive/network input reaches it. "Developer runs their own code" — by design, not a finding.
  Confidence: high.
  Axes examined: source of the expression/file callback arguments (traced end-to-end through `run.ex`); command reachability from dependency/archive/network flows | Not examined: none.

- **`process_load`/`--require` glob handling in `run.ex` (lines 190-223) — arbitrary code execution via wildcard file requires. — Upgraded from hedged assumption in first pass.**
  The first pass declined to audit this ("did not exhaustively audit `process_load`/`--require`"). I have now read it. `process_load/2` iterates the parsed `opts`; for each `{:require, value}` it calls `filter_patterns(value)` → `Path.wildcard(value)` filtered by `File.regular?`, then `require_runner.(filtered)` → `Code.require_file/1` (or `Kernel.ParallelCompiler.require/2` when `--parallel`). Both `require_file` and the parallel compiler execute the required files. The `value` originates solely from the `--require`/`-r`/`--parallel-require` switches parsed by `OptionParser.parse_head!` inside `profile.eprof`'s own `run/1` (line 144) — operator-typed argv. `Path.wildcard` is a filename glob over the operator-chosen pattern; the resulting files are the operator's own project files. `{:eval, value}` likewise routes the operator's `-e` string to the expression callback. No dependency/lockfile/archive/network data enters `process_load`. Executing operator-selected files is the task's contract, not a boundary crossing.
  Confidence: high.
  Axes examined: origin of `--require` pattern and `--eval` value (traced to this file's argv parse); the wildcard-expansion sink (`Path.wildcard` → `Code.require_file`); whether any non-operator source can populate `opts[:require]`/`opts[:eval]` | Not examined: none.

- **`Mix.Utils.parse_mfa/1` on `--matching` (line 174) — atom creation (CWE-770).**
  `parse_mfa` parses the string via `Code.string_to_quoted`, minting alias atoms. Executed in first pass: `parse_mfa("Totally.Bogus.Module.Never.Loaded")` → `{:ok, [Totally.Bogus.Module.Never.Loaded]}` (new atom); malformed input → `:error` cleanly, and `parse_opt` turns `:error` into a raising `Mix.raise` (line 178), not a silent crash. The `--matching` value is operator-supplied argv, not cross-boundary data, and atom creation is inherent to parsing a `Module.function/arity` pattern.
  Confidence: high.
  Axes examined: new-atom creation on well-formed input (ran); error path on malformed input (ran); argv provenance of the value | Not examined: none.

- **`:io.format(@format, to_print)` in `print_row/2` (line 338) — format-string / injection.**
  `@format` is a fixed literal (`"~-*s ~*s ~*s ~*s ~*s~n"`); per-row data (module/function names from `:eprof.dump()`) is passed only as `~s` arguments. Executed in first pass: `io:format("~-*s ~*s …", [7,"~w~n~s",…])` printed `~w~n~s` verbatim — `~` directives in data cells are never interpreted. No format-string injection even for an exotically named profiled module.
  Confidence: high.
  Axes examined: whether the format string is attacker-influenced (it is a module constant); whether data cells are interpreted as directives (ran) | Not examined: none.

- **`profile/2` programmatic API (line 205) — untrusted `opts`.**
  The caller supplies the 0-arity `fun` and a typed keyword list. `:matching` only narrows tracing (no sink), `:calls`/`:time` are integers used in comparisons, and `:sort` selects a sort function. Passing `:sort` a value outside the documented `:time | :calls` spec yields a `FunctionClauseError` in `sort_function/1` (no catch-all, lines 252-253), but that is caller misuse violating the `@spec` and only crashes — out of scope.
  Confidence: high.
  Axes examined: each opt's sink (matching/calls/time/sort); crash-vs-corruption on out-of-spec `:sort` | Not examined: none — but note this is a same-process programmatic API with no untrusted caller in scope.

- **Division by zero in `format_row`/`format_total` (lines 299-300, 308).**
  Guarded by `divide(_, 0) -> 0.0` (lines 319-320); all three call sites route through `divide/2`.
  Confidence: high.
  Axes examined: every division call site | Not examined: none.

## Risks / caveats

- **TOCTOU on the script path:** `Mix.Tasks.Run.run/5` checks `File.regular?(file)` (`run.ex:166`) then this task's callback does `File.read!(file)` (line 152). Confirmed the two-step check/read against `run.ex`. A local attacker who could swap the file between check and read could change what is read, but the path is operator-chosen and such an attacker already controls the working directory; `File.read!` on a now-non-regular target merely raises. Not a meaningful boundary crossing.
  Confidence: high (that this is not a boundary crossing).

- **Display-time robustness:** `format_row` → `String.to_charlist/1` (line 335) on `Exception.format_mfa/3` output could theoretically raise on a module whose atom name is invalid UTF-8 (constructible only by the profiled code itself, e.g. a dependency calling `binary_to_atom(_, :latin1)`). This is at most a crash of the profiler's own output on the developer's own loaded code — self-inflicted, not an attacker-across-a-boundary condition. I did not construct this case; it remains an acknowledged robustness caveat, not a finding, because the input source (the profiled program's own module names) is inside the operator trust boundary.
  Confidence: medium (that the only reachable source is operator-trusted code).

## Did I verify this?

- I read `run.ex` in full and traced the `--eval`, positional-file, and `--require` data flows from `profile.eprof`'s `run/1` argv parse (line 144) through `Mix.Tasks.Run.run/5` to their execution sinks. The first-pass hedge ("did not exhaustively audit `process_load`/`--require`") is now resolved: those sinks are operator-argv-driven with no dependency/archive/network input, promoted to a verified ruled-out entry above.
- The `parse_mfa`, `:io.format`, and `divide` ruled-out entries rest on code I executed in the first pass (new-atom creation, verbatim `~`-directive emission, zero-guard) and I re-confirmed the call sites against the current file.
- I did not re-execute the atom/format experiments this pass (the file bytes are unchanged per the sha256 header, and the relevant lines are identical), and I did not construct the invalid-UTF-8-atom display case, which stays a self-inflicted robustness caveat.

## Dropped in re-review

None. Every prior ruled-out entry and caveat is preserved; the only change is the `process_load`/`--require` hedge being upgraded from an unverified assumption to a verified ruled-out entry after reading `run.ex`.
