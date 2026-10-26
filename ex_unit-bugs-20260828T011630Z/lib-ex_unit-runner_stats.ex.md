# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/runner_stats.ex
<!-- sha256: dc2856415567c0ce6296a5024d8a3102fbb6b1b48b50ae80bc38f610a2c34409 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:38:15Z_

I've confirmed the behavior. Let me write up the report.

## Summary

`ExUnit.RunnerStats` correctly re-classifies a failed module's previously-passing tests as failures in its live counters, but fails to record those same tests in the on-disk failures manifest, so `mix test --failed` silently skips them.

## Bugfixes

#### `CSCAN-runner_stats-failed-module-manifest-omits-passed-tests`

- **Finding ID** — `CSCAN-runner_stats-failed-module-manifest-omits-passed-tests`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `runner_stats.ex:99`, in `handle_cast({:module_finished, %ExUnit.TestModule{state: {:failed, _}}}, state)` — the `Enum.reduce/3` body.
- **Class** — State / accumulator threading (a passed-through value has the wrong shape for its intended effect)
- **Impact** — When a module's `setup_all`/module-level teardown fails *after* its tests have already passed (tests emit `test_finished` with `state: nil`, then the module is marked `{:failed, _}`), those tests are correctly bumped into the `failures` counter for the on-screen summary — but they are **not** written into the failures manifest. On the next `mix test --failed`, these tests are not re-run, even though ExUnit reported them as failures. The user is told N tests failed but `--failed` re-runs fewer than N.
- **Evidence** — `verified`. The reduce calls `FailuresManifest.put_test(&1, test)` with `test.state == nil`. `FailuresManifest.put_test/2` (failures_manifest.ex:20-22) treats a `nil`-state test as *passing* and does `Map.delete(manifest, {module, name})` — i.e. it *removes* the test rather than recording it. Driving the GenServer through its documented event casts:

  ```
  $ /opt/elixir/bin/elixir /tmp/repro2.exs
  stats: %{total: 2, excluded: 0, skipped: 0, failures: 2}
  passing (nil-state) test recorded as failed in manifest? false
  failing test recorded in manifest? true
  ```

  The script sends `test_finished` for one passing (`state: nil`) and one failing test in module `SomeMod`, then `module_finished` with the module state `{:failed, _}`, then `suite_finished` (which writes the manifest). Result: the counter shows `failures: 2` (both counted), but only the genuinely-failing test lands in the manifest; the invalidated passing test is absent, so `--failed` would skip it.

  The intent of the reduce is unambiguous — it bundles the manifest update with `failures + 1` / `passed - 1` for exactly these tests, and the comment says *"we should only invalidate the successful tests"*. But `put_test` on a `nil`-state test is the *opposite* of invalidating: for a test that `test_finished` already deleted from the manifest, this second call is a redundant no-op, so the manifest line accomplishes nothing.

- **Suggested fix** — Give the test a failed state before recording it, so `put_test` takes its `:failed`/`:invalid` branch. Minimal change at line 99:

  ```elixir
  |> Map.update!(:failures_manifest, &FailuresManifest.put_test(&1, %{test | state: test_module.state}))
  ```

  (`test_module.state` is `{:failed, _}`, which `put_test` records via `Map.put`.) This keeps the manifest consistent with the counters and with what the summary reports.

## Risks / caveats

- The exact-effect scenario (module marked `{:failed, _}` while its member tests still hold `state: nil`) arises specifically when tests pass and *then* a `setup_all`/module teardown or `on_exit` fails. If `setup_all` fails *before* tests run, the tests carry `{:invalid, _}` rather than `nil`, the `is_nil/1` filter excludes them, and this handler is a no-op — so the impact is confined to the post-pass-teardown-failure path. That path is real and reachable through ordinary test authoring.
- I could not diff against upstream Elixir (no VCS in the tree; `WebFetch` was not permitted), so the "intended behavior" argument rests on the in-file comment, the coupling with the counter updates, and the fact that the manifest call is otherwise a pure no-op — not on an upstream reference. The *observable* behavior (passed-then-invalidated tests missing from the manifest) is verified by execution.
- The reproduction ran against `/opt/elixir` (the environment's built-from-source toolchain); `ExUnit.RunnerStats`, `ExUnit.FailuresManifest`, `ExUnit.Test`, and `ExUnit.TestModule` resolved from that build, which matches the source under review.

### Ruled out

- **`increment_failure_counter/2` returns the pre-increment value, not the new total.** Not a bug: the sole caller, `Runner.process_max_failures/3` (runner.ex:571-578), binds it as `previous` and computes `previous + bump` deliberately; `@spec ... :: non_neg_integer` is satisfied by the pre-increment value. Verified by reading the caller.
- **`handle_cast(_, state)` / missing-message crash (HANDLE_CAST_TOLERATES_UNKNOWN).** A catch-all `handle_cast(_, state)` (line 107) and the `is_binary(file)` guards on `suite_started`/`suite_finished` mean unknown casts and a `nil` manifest path are silently ignored, not crashed. Present and correct.
- **`increment_status_counter/2` fall-through (line 127) silently dropping a status.** The default clause leaves `state` unchanged for unrecognized state shapes, which could under-count — but every state shape `Runner.failed/3`/`prepare_tests` can produce (`nil`, `{:skipped,_}`, `{:excluded,_}`, `{:failed,_}`, `{:invalid,_}`) is matched by an earlier clause. Reaching the fall-through requires an undocumented `test.state` shape (internal-contract misuse), which is out of scope. Verified by enumerating the clauses against the producing sites.
- **`passed` counter going negative in the `module_finished` handler (`&(&1 - 1)`).** Only tests with `state: nil` are decremented, and a `nil`-state test always emitted `test_finished` first (incrementing `passed`), so the decrement is balanced. No reachable path drives `passed` below zero.

