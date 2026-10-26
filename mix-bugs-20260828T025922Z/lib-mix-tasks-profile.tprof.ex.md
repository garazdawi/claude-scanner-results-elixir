# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.tprof.ex
<!-- sha256: 5f40fab1f7ddf46c7305b041acbc401db966c72417cd7474b44b0a846837d9f5 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.tprof.ex
<!-- sha256: 5f40fab1f7ddf46c7305b041acbc401db966c72417cd7474b44b0a846837d9f5 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`profile.tprof.ex` implements the `mix profile.tprof` task and the public `Mix.Tasks.Profile.Tprof.profile/2` entry point; the profiling/printing paths for all three types (`time`, `memory`, `calls`) work correctly, but the command-line argument parser diverges from its sibling tasks in a way that breaks passing switch-form options to a profiled script. In this re-review I read the two neighbour profiler tasks and `Mix.Tasks.Run` (`run/5`) end-to-end, and re-ran the parser comparison, converting the two first-pass hedges into verified facts. The finding stands and is now fully verified across the call chain.

## Bugfixes

#### `CSCAN-profile.tprof-parse-not-parse-head`
- **Finding ID:** `CSCAN-profile.tprof-parse-not-parse-head`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `profile.tprof.ex:206` (`parse!/1`, called from `run/1` at line 192)
- **Class:** Contract violation (documented "syntax similar to `mix run`" not honored) — argument-parsing defect at the task `run/1` boundary.
- **Impact:** A user who runs the documented form `mix profile.tprof my_script.exs <args>` and includes any switch-form token in the script's arguments (e.g. `mix profile.tprof my_script.exs --verbose`) gets a hard failure — `** (OptionParser.ParseError) ... Unknown option --verbose` — instead of the script being profiled with `--verbose` forwarded to it via `System.argv`. `mix profile.eprof` and `mix profile.fprof` accept the same invocation because they use `OptionParser.parse_head!`. The tprof task is the odd one out and cannot profile any script that takes its own flags.
- **Evidence:** `verified` (re-executed this pass; call-chain read this pass).
  - Parser divergence re-run this pass (locally, against OTP's `OptionParser`):
    - `OptionParser.parse_head!(["my_script.exs", "--verbose", "x"], …)` → `{[], ["my_script.exs", "--verbose", "x"]}` — stops at the first non-switch, leaving the remainder for the script.
    - `OptionParser.parse!(["my_script.exs", "--verbose", "x"], …)` → **raises** `1 error found! --verbose : Unknown option`.
    - `OptionParser.parse!(["my_script.exs", "arg1", "arg2"], …)` → `{[], ["my_script.exs", "arg1", "arg2"]}` — positional-only args are unaffected.
  - Sibling tasks confirmed by reading: `profile.eprof.ex:144` and `profile.fprof.ex:149` both call `OptionParser.parse_head!(args, aliases: @aliases, strict: @switches)`; only `profile.tprof.ex:206` (via the `parse!/1` helper) uses `OptionParser.parse!`.
  - **End-to-end impact confirmed by reading `Mix.Tasks.Run.run/5` (`run.ex:120-174`)** — *upgraded from the first-pass "out of scope" hedge*: `run/5` derives the script file and the script's `argv` from `head` (`run.ex:135-141`: `{_, [head | tail]} -> {head, tail}`) and then calls `System.argv(argv)` (`run.ex:143`), so switch-form tokens intended for the script must survive in `head` to reach `System.argv`. Because tprof's `parse!/1` (`OptionParser.parse!`) raises on any unknown switch *before* `run/5` is ever called (`run/1` at line 192 runs `parse!` first), the script never runs. The defect is therefore both determined at the `parse!/1` boundary and observable end-to-end, not merely a local parsing quirk.
- **Suggested fix:** In `parse!/1` change `OptionParser.parse!(args, …)` to `OptionParser.parse_head!(args, …)`, matching the eprof/fprof tasks.

## Risks / caveats

- I verified the three profiling types, `--report total`/`process`, `--sort per_call`, `--matching`, and the incompatible-option guards (`get_filter_value!`, the `sort_by` `cond`) all behave as documented by running them against the OTP 29 `:tprof` built from this source tree; none crashed or misprinted (the mojibake on `µS/CALL` in my runs is a latin1-locale artifact of the sandbox, not a code defect, and matches eprof/fprof; the same latin1 warning appeared in this pass's re-run).
- **Resolved (was a first-pass hedge):** The claim that `mix profile.tprof my_script.exs …` is meant to forward the script's own options to it is now verified, not assumed. `Mix.Tasks.Run.run/5` writes the script's tail args into `System.argv` (`run.ex:143`), the tprof/eprof/fprof moduledocs all state "syntax similar to the `mix run` task" and show `my_script.exs arg1 arg2 arg3`, and the two sibling tasks use `parse_head!` (read this pass). There is no remaining ambiguity about intent — forwarding is the designed behavior and tprof breaks it.

### Ruled out
- **`:report` passed as `:process`/`:total` to `:tprof.inspect`** — I suspected the atom should be `:per_process`, but verified by execution `tprof:inspect(Prof, process, calls)` and `tprof:inspect(Prof, total, calls)` both succeed on OTP 29 (a `garbage_atom` raises `function_clause`), so `:process`/`:total` are the correct atoms. Not examined: none.
- **`calls` type not forced to `report: :total`** — docs say report is always total for `calls`; code leaves the default `:process`. Verified harmless by execution: `call_count` tracing is global, so `:tprof.inspect` returns the single `all` key either way and `print_pid_row(:all)` prints "over all processes". Not examined: none.
- **`sort_by` default `:measurement` for `calls` type** — verified by execution that `:measurement`, `:calls`, `:measurement_per_call`, `:module` are all accepted keysorts by `:tprof.inspect`. Not examined: none.
- **`column_lengths`/`print_row` column-count mismatch for `calls` type (3-col header vs 5-col rows)** — verified: `Enum.zip_with`/`Stream.zip` truncate to the shorter list, so only the first 3 columns (`#`, `CALLS`, `%`) are formatted with `@call_format`; output is correct. Not examined: none.
- **`divide/2` by zero in `format_total`** — guarded by the `divide(_, 0) -> 0.0` clause; reached when all results are filtered out. No crash. Not examined: none.

### Did I verify this?

- **`parse!/1` divergence from siblings** — Yes. Re-ran `OptionParser.parse_head!` vs `OptionParser.parse!` on `["my_script.exs", "--verbose", "x"]` and on positional-only input this pass; read `profile.eprof.ex:144` and `profile.fprof.ex:149` confirming `parse_head!`.
- **End-to-end forwarding to `System.argv`** — Yes. Read `Mix.Tasks.Run.run/5` (`run.ex:120-174`); confirmed `head`→`argv`→`System.argv(argv)` and that tprof's `parse!` raises before `run/5` is reached. (First-pass "out of scope for this file" hedge removed.)
- **Intent that script switches are forwarded** — Yes. Cross-checked the three moduledocs and `run.ex` behavior; no longer an assumption.
- **Profiling/printing paths and ruled-out items** — Yes, by execution against OTP 29 `:tprof` (carried over from first pass; nothing in those paths changed and no axis was left untried).

### Dropped in re-review

None. Every first-pass finding and ruled-out entry is preserved; the two "Risks / caveats" hedges were upgraded to verified facts (via reading `run.ex`, `profile.eprof.ex`, `profile.fprof.ex` and re-executing the parser comparison) rather than dropped or downgraded.
