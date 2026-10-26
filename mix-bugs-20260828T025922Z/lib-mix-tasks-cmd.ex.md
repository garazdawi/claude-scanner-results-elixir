# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/cmd.ex
<!-- sha256: 1a5ab0471bb55ca78404397ee19b809c25ba8e9cff36ab281c7ec4788a683498 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:40:15Z_

## Summary

`mix cmd` (`Mix.Tasks.Cmd.run/1`) parses options and shells out to the given command; its argument-shaping and option handling are correct, but the non-zero-exit path aborts using a bare-integer `exit/1` that prints a spurious crash dump and drops the sub-command's real exit status.

## Bugfixes

#### `CSCAN-cmd-bare-integer-exit-status`
- **Finding ID:** `CSCAN-cmd-bare-integer-exit-status`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `lib/mix/lib/mix/tasks/cmd.ex:135`, `run/1` (the `status -> exit(status)` clause)
- **Class:** Error-handling asymmetry (abnormal-exit vs. clean shutdown)
- **Impact:** When the invoked command exits non-zero, `mix cmd` calls `exit(status)` with a bare integer. Elixir's CLI treats that as an *abnormal* process exit: it prints an Elixir crash dump (`** (exit) 3` plus a stacktrace pointing at `cmd.ex:135`) and then halts the OS process with exit code **1** — not with the command's actual status. Two concrete costs: (1) every failing `mix cmd` invocation (linters, `make`, tests-via-cmd, umbrella per-app commands) emits a confusing internal stacktrace that looks like a Mix bug rather than a normal command failure; (2) scripts/CI that branch on the propagated exit code always see `1`, so the real status (e.g. 2 for "lint warnings", 3, …) is lost. Every other abort site in Mix (`test.ex`, `run.ex`, `compile.all.ex`, `cli.ex`, …) uses `exit({:shutdown, code})` precisely to avoid both problems; `cmd.ex` is the lone outlier.
- **Evidence:** `verified`.
  - Reproduction via the public task:
    ```
    $ mix cmd sh -c "exit 3"
    ** (exit) 3
        (mix 1.21.0-dev) lib/mix/tasks/cmd.ex:135: Mix.Tasks.Cmd.run/1
        (mix 1.21.0-dev) lib/mix/task.ex:502: anonymous fn/3 in Mix.Task.run_task/5
        (mix 1.21.0-dev) lib/mix/cli.ex:129: Mix.CLI.run_task/2
        /opt/elixir/bin/mix:7: (file)
    EXIT_CODE=1
    ```
    The sub-command exited 3, but Mix printed a stacktrace and the shell saw exit code **1**.
  - Direct contrast of the two exit forms:
    ```
    $ elixir -e 'exit({:shutdown, 3})'; echo CODE=$?   # -> CODE=3, no stacktrace
    $ elixir -e 'exit(3)'; echo CODE=$?                 # -> ** (exit) 3 + stacktrace, CODE=1
    ```
  - `grep` confirms `cmd.ex:135` is the only `exit(<bare int>)` in `lib/mix`; all sibling abort sites use `exit({:shutdown, code})`.
- **Suggested fix:** Replace `status -> exit(status)` with `status -> exit({:shutdown, status})`, matching the rest of Mix; this halts cleanly with the sub-command's status and suppresses the crash dump.

## Risks / caveats

- The reviewed file is byte-identical to the installed `1.21.0-dev` tree (`diff` reported IDENTICAL), so the reproductions above exercise exactly the code under review.
- The exit-code test was run on a single-project (non-umbrella) layout; I did not re-run it inside an umbrella, but the `exit/1` call site is shared by both paths, so the behaviour is the same.
- Windows-only path handling (`String.contains?(path, ["/", "\\"])` / `Path.type`) could not be exercised on this Linux sandbox; I reasoned about it but did not run it. I found nothing wrong there.

## Ruled out

- **`String.to_unsafe_atom/1` on `--app` values (line 109) — atom-table growth.** The `--app` switch is operator-supplied CLI input, is explicitly deprecated with an `IO.warn`, and the values are only compared against `Mix.Project.config()[:app]`; this is documented, bounded-by-operator behaviour, not a reachable defect from untrusted data.
- **`--cd` combined with the relative-path expansion branch (lines 124–125, 131).** Verified by reasoning through cases: a slash-containing relative command is expanded to an absolute path *within* `cd`, and the command still runs with cwd=`cd`, which is self-consistent; a bare command name (no slash) is left untouched and resolved via `PATH` with cwd=`cd`, matching the documented `mix cmd --cd third-party make` example.
- **`OptionParser.parse_head!` flag handling.** `parse_head` stops at the first non-switch, so flags after the command name are correctly forwarded to the underlying command and only `--cd/--shell/--app` before it are consumed — matches the documented "pass the option before the command name" contract.
- **`Mix.Task.reenable("cmd")` skipped on the failure path.** When the command fails, `exit` aborts the run, so not re-enabling the task is harmless; on success (`0 -> :ok`) the re-enable runs as documented.
- **Passing a `{path, rest}` tuple (vs. a string) to `Mix.shell().cmd/2`.** Confirmed the shell contract accepts both: `Mix.Shell.cmd/3` dispatches `{command, args} -> System.cmd(...)` and `binary -> System.shell(...)`, and `Mix.Shell.IO.cmd/2` forwards the value unchanged.

