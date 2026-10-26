# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.fprof.ex
<!-- sha256: 8477479d341eff244b5aa10b87bc05264fdccff85ac2feb48b3cb026750cd800 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T05:02:43Z_

## Summary

`Mix.Tasks.Profile.Fprof` wraps Erlang's `:fprof` behind the `mix profile.fprof` task and a public `profile/2` function; after exercising every documented option and both entry points against the source-built toolchain, I found no reachable defect in program behaviour.

## Bugfixes

No findings.

## Risks / caveats

- I verified the happy paths and every documented CLI option by running the task and the public `profile/2` against the in-tree Elixir/OTP 29 build: default output, `--details`, `--callers`, `--sort own`, `--trace-to-file`, `--no-warmup`, an invalid `--sort` value, a script-file + argv invocation, and process-spawning code that exercises the `spawned_by`/`spawned_as`/`initial_calls` branches. All produced correct output and correct return values.
- Paths I reasoned about but could not drive to failure through the public API (they require malformed `:fprof` analysis output, which is not caller-controllable): the `{:ok, term} = :erl_parse.parse_term(...)` and `{term, leftover} = next_term(...)` destructures in `print_output`/`next_term`, and the `else :ok ->` clause in `profile_and_analyse`. These would raise `MatchError`/`CaseClauseError` only if `:fprof.analyse` emitted non-term or non-`:ok` output, which a well-behaved caller cannot induce.

### Ruled out

- **`--trace-to-file` temp-file leak.** Verified the trace file is removed on the normal path: `ls /tmp/fprof_trace_*` counted 0 before and 0 after a run. `File.rm/1` sits in the `after` block, so it also runs on the analyse-error path; and `trace_file` vs its `Path.expand`ed form resolve to the same absolute path (`System.tmp_dir!()` is absolute), so `File.rm` targets the created file.
- **`profile/2` return value (`@spec … :: result`).** Verified `{:val, 99}` and `:hello` and `42` returned unchanged across trace and non-trace branches and with `details`/`callers`/`sort` set — matches the documented "Returns the return value of `fun`."
- **`parse_opt` sort handling.** Verified `--sort own` sorts by own, `--sort acc`/absent defaults to acc, and `--sort bad` raises the documented `** (Mix) Invalid sort option: bad` rather than crashing. Non-sort options pass through untouched.
- **`next_term`/`Stream.unfold` termination.** Verified the stream terminates cleanly (no hang, no crash) across all option combinations; the `_ -> nil` clause absorbs the `{:more, _}` returned by `:erl_scan.tokens` at end-of-data, which is the intended stop condition.
- **`print_analysis_result` clause coverage.** Verified all three clauses (PID-entry list, `--callers` 3-tuple, plain 4-tuple) match real `:fprof` output including process-detail sections; `function_text/1`'s 3-tuple clause correctly formats both `{m,f,arity}` and `{m,f,args}` shapes via `Exception.format_mfa`.
- **Double execution of profiled code.** The warmup run of `fun` before profiling is documented (`--no-warmup` disables it), not a defect.

