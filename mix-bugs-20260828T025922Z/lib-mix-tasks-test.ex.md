# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/test.ex
<!-- sha256: 57b020574f4d00607063d1e81dfda0db72ced3fa1231feb57b1d59910c358ac8 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/test.ex
<!-- sha256: 57b020574f4d00607063d1e81dfda0db72ced3fa1231feb57b1d59910c358ac8 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`Mix.Tasks.Test` (the `mix test` task) is a large orchestration module whose only documented entry point is `run/1`. On this second pass I resolved the one hedge the first-pass reviewer flagged ("Cross-module behaviour not verified end-to-end / outside this file") by reading each named callgraph neighbour and by executing the two divergence/partition axes on the local Erlang/OTP 29 + Elixir 1.21.0-dev toolchain. Every cross-module contract the result-`cond` and helper paths depend on is now confirmed against the collaborator's `@spec` and implementation, and both ruled-out arithmetic entries had their previously-unexamined axes actually driven through code. I still find **no genuine reachable bug**. The file remains byte-for-byte identical to the shipped source (`diff` exit 0 in the prior pass; sha unchanged).

## Bugfixes

No findings.

## Risks / caveats

- **Provenance:** file matches the source the local `elixir`/`mix` was built from; any bug would be a latent upstream defect, and this file is heavily exercised by the Elixir suite.

## Ruled out

- **Cross-module contract of the result-`cond` dispatch and helpers — now verified by reading neighbours (resolves the first-pass "outside this file" hedge).**
  - `CT.require_and_run/4` (`Mix.Compilers.Test`, `/opt/elixir/lib/mix/lib/mix/compilers/test.ex:28-118`) returns **exactly two shapes**: `:noop` (empty test set, or `--profile-require time`) and `{:ok, Map.put(results, :warnings?, warnings?)}` where `results = ExUnit.await_run(task)` carries `:failures`, `:excluded`, `:total`. The `else` block's two clauses — `{{:ok, %{excluded:, failures:, warnings?:, total:}}, helper_warned?}` (test.ex:715) and `{:noop, _}` (test.ex:748) — cover both and nothing else. On compile `{:error, _, _}` the collaborator calls `exit({:shutdown, 1})` rather than returning a third shape, so no `CaseClauseError` is reachable here. Verified by reading, not assumed.
  - `ExUnit.Filters.parse_paths/1` (`/opt/elixir/lib/ex_unit/lib/ex_unit/filters.ex:30-42`), `@spec … :: {[String.t()], ex_unit_opts}` — `Enum.map_reduce` yields one parsed path per input element. `relative_app_file_exists?/1` (test.ex:568) matches `{[file], _}` against a single-element input list; that pattern is total. Verified.
  - `ExUnit.Filters.failure_info/1` (filters.ex:163-166), `@spec … :: {MapSet.t, MapSet.t} | :all` — `manifest_opts/1` (test.ex:1022-1028) handles precisely `{allowed_files, failed_ids}` and `:all`. Both branches present; total. Verified.
  - `ExUnit.Filters.fail_all!/1` (filters.ex:174-177), `@spec … :: :ok` — called only in the `catch` before re-raising (test.ex:712); return value discarded. Verified.
  - Coverage `tool.start/2`: default `Mix.Tasks.Test.Coverage.start/2` (`/opt/elixir/lib/mix/lib/mix/tasks/test.coverage.ex:194-214`) **always returns a zero-arity fn** (never `nil`). The `@moduledoc` (test.ex:410-413) permits a custom tool to return `nil`, and the call site guards with `cover && cover.()` (test.ex:718), so a `nil`-returning tool is handled. Verified by reading both sides.
  - _Axes examined: return-shape totality of all five collaborators (read each `@spec` + every return path in the implementation) | Not examined: none of the shape axes — behavioural end-to-end run under a live `mix.exs` project (no project tree present) is not a distinct return-shape axis, since every reachable return value of each collaborator is enumerable from its source._

- **`String.trim_leading(file, current_app_path)` (test.ex:555) vs `String.replace_prefix`.** Executed both on the Elixir toolchain across the contiguous-repeat axis. Divergence occurs **only** when the file path repeats the `apps/<app>/` prefix contiguously at the front — e.g. `apps/foo/apps/foo/test/x.exs` → `trim_leading` yields `test/x.exs` while `replace_prefix` yields `apps/foo/test/x.exs` (DIFFER=true). For a normal path (`apps/foo/test/x_test.exs`), an app-name substring collision (`apps/foo/foo/test/x.exs`), and a non-boundary prefix (`apps/fooapps/foo/...`) the two are byte-identical (DIFFER=false, all confirmed by running). The diverging input requires an app named `foo` to physically contain a directory `apps/foo/` inside its own tree — not a realistic umbrella layout. Not a real defect.
  - _Axes examined: contiguous-prefix-repeat, app-name substring collision, non-boundary substring, deep triple-repeat — all pushed through both functions and compared (see executed output) | Not examined: none._

- **`filter_by_partition/3` round-robin `rem(index, total) == partition-1` (test.ex:1054-1056) plus the `1..total//1` / `total > 1` / fallback guards.** Executed: partitioning 10 files across 3 partitions, the union of all partitions equals the sorted input with no duplicates (each file assigned exactly once — confirmed `union==input sorted? true`, `no dup? true`). The range guard `partition in 1..total//1` correctly rejects out-of-range values (`2 in 1..1//1 → false`, `0 in 1..3//1 → false`), driving the `_ -> raise_with_shell` clause; the third `filter_by_partition/3` clause rejects non-positive `total`. Matches the documented "must be a number greater than zero" contract.
  - _Axes examined: coverage/no-gap, no-overlap, upper-bound reject (part=total+1 collapse via `1..1`), lower-bound reject (0), non-positive total fallback — all executed | Not examined: none._

- **`partition = partitioned? && System.get_env("MIX_TEST_PARTITION")` → `export` (test.ex:618-622).** Yields `false` when coverage runs unpartitioned without `--export-coverage`; `false` and `nil` are equivalent under the coverage tool's `if name = opts[:export]` truthiness test (test.coverage.ex:204), so no wrong export file is written. Confirmed against the collaborator's `start/2` body.
  - _Axes examined: `false` vs `nil` truthiness at the consumer's `if name = opts[:export]` | Not examined: none._

- **Warnings-as-errors exit status.** `exit_status + 1` (test.ex:730), default `exit_status` = 2 (test.ex:1085), giving default `3` — matches the documented contract at test.ex:222-226. Structural; no ambiguous axis.

- **`@moduledoc` executable-documentation.** Contains only shell (`$ mix test …`) examples, no `iex>` doctests; nothing executable to fail.

## Did I verify this?

- Read the full target file (`test.ex`, 1109 lines).
- Read all four named callgraph neighbours in full/in-part: `ExUnit.Filters` (`filters.ex`), `Mix.Compilers.Test` (`compilers/test.ex`), and `Mix.Tasks.Test.Coverage.start/2` (`test.coverage.ex:194-214`) — confirming the `@spec`/return shapes the result-`cond`, `relative_app_file_exists?`, `manifest_opts`, and coverage setup depend on. This is option (a): the "outside this file" hedge is now backed by the neighbours' source, not by assumption.
- Executed two axes on the live toolchain (Erlang/OTP 29, Elixir 1.21.0-dev): the `trim_leading`/`replace_prefix` divergence table and the partition round-robin coverage/guard checks. Output cited above.
- No files modified.

## Dropped in re-review

- **Dropped the first-pass caveat "Cross-module behaviour not verified end-to-end … reasoned about these from their call sites … did not drive them through a real project run."** New evidence: I read each collaborator's source and enumerated every reachable return value, which is what that caveat said it had not done. Every return shape consumed by this file is total against its producer, so the caveat's residual doubt is resolved without needing a live project run (a project run would exercise the same enumerable return paths). No finding is being suppressed — the underlying claim is confirmed correct, so it moves from "unverified caveat" to a verified Ruled-out entry rather than being upgraded to a finding.

No prior finding existed to preserve (the first pass reported none), and no hedged assumption required upgrading to a low-confidence finding, because each was verifiable by reading the neighbour source or by executing the axis.
