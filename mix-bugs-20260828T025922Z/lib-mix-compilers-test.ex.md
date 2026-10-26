# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/compilers/test.ex
<!-- sha256: f8db9b96a37e6fc18b8082772d87f33dad53a9d2bfadc7f13179ad8ea940dcd6 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:11:07Z_

## Summary

`Mix.Compilers.Test` requires and runs test files, tracking staleness via an agent-backed manifest. The logic is mostly sound; the one real defect is a missing agent cleanup on the `profile_require` code path.

## Bugfixes

#### `CSCAN-test-profile-require-agent-leak`
- **Finding ID:** `CSCAN-test-profile-require-agent-leak`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `/Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/compilers/test.ex:75-77` (`require_and_run/3`, `profile_require` cond branch)
- **Class:** Resource leak
- **Impact:** When a caller passes both `stale: true` and `profile_require: "time"`, `set_up_stale/3` starts a linked stale-manifest `Agent` (`Agent.start_link`, line 167). If the stale computation yields a non-empty `test_files_to_run`, the `cond` dispatches to the `profile_require == "time"` branch, which returns `:noop` **without** calling `agent_stop(stale_manifest_pid)`. The other two branches (empty test set, and the normal run) both stop or never create the agent; this branch leaks it. The agent survives (linked, normal exits are ignored by a non-trapping GenServer) until the owning Mix process dies, and the manifest is also never refreshed. It is a single, bounded leak per invocation, not unbounded accumulation.
- **Evidence:** `asserted`. By inspection: `set_up_stale` (line 167) creates the pid; the `profile_require` branch (lines 75-77) has no `after`/`agent_stop`, unlike the `true` branch which stops it in its `after` (line 115). Reproducing end-to-end requires a full `mix test --stale --profile-require time` project run, which I could not stand up in the sandbox, so this is reasoned from the control flow rather than executed.
- **Suggested fix:** Stop the agent in the profile branch, e.g. wrap the require in `try ... after agent_stop(stale_manifest_pid) end`, mirroring the `true` branch.

## Risks / caveats

- The single finding is `asserted`, not `verified`: exercising it needs a real Mix project driven with `--stale --profile-require time`, which the isolated file review could not set up. The reachability rests on the assumption that the `mix test` task permits combining `--stale` and `--profile-require`; if the task layer rejects that combination upstream, the branch is unreachable and the finding is moot. I did not have the task module in view to confirm.
- I traced the `nil` vs `[]` third-tuple-element from `set_up_stale` and confirmed the `nil`-callbacks case only reaches the `test_files == []` branch (safe). The `exit({:shutdown, 1})` is correctly caught and re-raised with the runner task shut down, and the `Agent.cast` write-then-`Agent.stop` ordering preserves the manifest write. These were reviewed and ruled out below.

### Ruled out
- **`nil` `parallel_require_callbacks` crashing `shared_require_options ++ parallel_require_callbacks`** — `set_up_stale` returns `nil` callbacks only together with `test_files == []`, which routes to the first `cond` branch; the `++` on line 86 is only reached when callbacks is a proper list. No crash.
- **`exit({:shutdown, 1})` bypassing task shutdown** — the surrounding `catch kind, reason` catches exits, runs `Task.shutdown(task)`, and re-raises; cleanup is correct.
- **Lost manifest write from async `Agent.cast` before `agent_stop`** — `Agent.stop/3` is synchronous and drains the mailbox in order, so the cast'd `write_manifest` runs before termination.
- **`write_manifest/1` clause ordering (`%{}` pattern matching any map)** — the empty-map guard clause precedes the general `= %{}` clause, so empty maps delete the manifest and non-empty maps are written; ordering is correct.
- **`read_manifest/0` returning `%{}` on corrupt/old manifest** — the `rescue` intentionally degrades to an empty map (treat-as-stale), consistent with how downstream `mtimes`/`update_stale_sources` consume a map.

