# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/loadpaths.ex
<!-- sha256: fe254fe901e4501180f0cb1841320969963a996c2c28607f02b037b7267e8f93 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:56:02Z_

## Summary

`Mix.Tasks.Loadpaths` is a small delegating build task that runs `deps.loadpaths`, optionally erases the app build on an SCM/version-lock mismatch, and prepends the compile path; its control flow is total and I found no genuine, reachable bugs.

## Bugfixes

No findings.

The file's logic is straightforward and I confirmed the one non-trivial data-flow — the `case` over `Mix.Dep.ElixirSCM.read/0`:

- `Mix.Dep.ElixirSCM.read/0` (`elixir_scm.ex:27-40`) returns exactly `{:ok, vsn, scm, lock, deps}` (a 5-tuple) or `:error`. The `case` in `load_project/1` (`loadpaths.ex:61-74`) matches `{:ok, old_vsn, old_scm, _deps_lock, _deps}` guarded by `old_vsn != vsn or old_scm != scm`, with a `_ -> :ok` catch-all. Every possible return value — the `:error` atom, and the `{:ok, ...}` tuple that fails the guard — falls to the catch-all, so the match is total. No `CaseClauseError` is reachable.
- `vsn` is `{System.version(), :erlang.system_info(:otp_release)}`, structurally identical to the tuple stored by `ElixirSCM.update/4` (`elixir_scm.ex:20`), so the `old_vsn != vsn` comparison is well-formed and the rescue fallback `{"1.0.0", ~c"17"}` deliberately never equals a live version, forcing recompilation as intended.
- `run/1` guards the `load_project` call with `if config[:app]`, and the `--from-mix-deps-compile` membership test uses `in`/`not in` against the raw `args` list correctly.

## Risks / caveats

- The review is confined to `loadpaths.ex`. Correctness of the erase-on-mismatch behaviour depends on `Mix.Project.app_path/consolidation_path/compile_path` and `Mix.Dep.ElixirSCM` returning their documented shapes; I read `ElixirSCM` to confirm the tuple contract but took the `Mix.Project` path helpers as trusted (they are long-standing public API).
- I did not execute a live `mix` scenario driving `run/1`, because the only candidate defect (case-clause totality) is settled by static inspection of both the producer and consumer of the value; there was no behavioural ambiguity left to resolve by running. The totality claim above is grounded in the full enumeration of `read/0`'s return values in `elixir_scm.ex`, not a partial read.

