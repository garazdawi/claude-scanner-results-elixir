# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/parallel_compiler.ex
<!-- sha256: edec4c33b1f03ee5566eb60b2651f3dc11f6ce23784937964dec6bcde4189f9f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:31:57Z_

## Summary
The reviewed file is byte-identical to the pristine Elixir 1.21.0-dev standard-library source (`diff` against `/opt/elixir/...` returned no differences), and the public compile/require/deadlock/deprecated paths all behave correctly when exercised; the only concrete defect found is an uninformative crash in `pmap/2` on an empty collection outside a compilation context.

## Bugfixes

#### `CSCAN-parallel_compiler-pmap-empty-collection-matcherror`
- **Finding ID** — `CSCAN-parallel_compiler-pmap-empty-collection-matcherror`
- **Severity** — Low
- **Confidence** — high
- **Location** — `parallel_compiler.ex:116`, `pmap/2`
- **Issue** — `pmap/2` maps `inner_async/1` over the collection first, and only afterwards reads `{compiler_pid, file_pid} = :erlang.get(:elixir_compiler_info)`. When the collection is **non-empty** and there is no active compiler, `inner_async/1` raises a clear `ArgumentError` ("cannot spawn parallel compiler task because the current file is not being compiled/required"). When the collection is **empty**, `inner_async/1` is never invoked, so control reaches line 116 where `:erlang.get(:elixir_compiler_info)` returns `:undefined`, and the tuple match fails with a bare `MatchError`.
- **Class** — Missing clause / non-total function (uninformative crash on a valid argument).
- **Impact** — A caller who passes an empty list to the public `pmap/2` outside a compilation gets `** (MatchError) no match of right hand side value: :undefined`, which says nothing about the actual precondition (must be run inside compilation), whereas every non-empty input to the same misuse gets a descriptive `ArgumentError`. Inconsistent, undebuggable error for a perfectly valid argument shape (`[]`).
- **Evidence** — `verified`. Ran:
  ```
  Kernel.ParallelCompiler.pmap([1,2], fn x -> x end)
  # rescued: ArgumentError: cannot spawn parallel compiler task because the current file is not being compiled/required
  Kernel.ParallelCompiler.pmap([], fn x -> x end)
  # rescued: MatchError: no match of right hand side value: :undefined
  ```
  (Confirmed separately that `pmap([], fun)` *inside* a real compilation correctly returns `[]`, so the defect is specifically the empty-collection-outside-compilation path.)
- **Suggested fix** — Read and validate the compiler context before mapping, e.g. `case :erlang.get(:elixir_compiler_info) do {compiler_pid, file_pid} -> ... ; :undefined -> raise ArgumentError, "..." end`, or hoist the existing `inner_async/1` `:undefined` check so the empty-collection path raises the same `ArgumentError`.

## Risks / caveats
- The file is an exact copy of the shipped upstream compiler front-end (`diff` clean against the built `/opt/elixir` tree), so it is mature, heavily exercised code; I found no correctness defect in the core compile/require orchestration.
- The `pmap/2` finding is a misuse-adjacent error-quality issue (Low): calling `pmap` outside a compilation is itself unsupported, so the impact is limited to a confusing error message rather than wrong compilation output.
- I did not attempt to construct exotic async-only deadlock states (reachable only through the deprecated `async/1` plus hand-crafted internal message sequences); the `handle_deadlock/2` `Enum.max([])`/`Process.info(dead_pid, ...)` theoretical edges were not reproducible through the normal `compile/2` deadlock path, which I exercised and confirmed produces correct diagnostics.

### Ruled out
- **Deadlock reporting crash (`handle_deadlock` `Enum.max`/`Process.info` nil)** — Basis `verified`: compiled two files with a circular struct dependency via the public `compile/2`; the deadlock was detected and reported as well-formed `:error` diagnostics with no crash, so the real deadlock path is sound.
- **`compile/2` without `return_diagnostics` conversion path (`spawn_workers/3` lines 333–339)** — Basis `verified`: ran a compile that emits a warning without `return_diagnostics`; it warned as expected and returned the documented deprecated `{:ok, [atom], [{file, position, message}]}` shape without error.
- **`each_file/1` arity dispatch and default (`fn _, _ -> :ok end` default, arity-1 user callback)** — Basis `verified`: normal `compile/2` runs (warning-producing and clean) invoke `state.each_file.(file, lexical)` without `FunctionClauseError`, confirming the arity-1 wrapping and arity-2 default both resolve.
- **Time-unit arithmetic in `profile/3`, `cycle_timing/2`, `to_padded_ms/1`, `maybe_warn_long_compilation/2`** — Basis `asserted`: each conversion pipes the correct value as the first argument and uses matching units (`:timer.tc` µs → `div(_,1000)` ms; `send_after`/threshold in ms; `convert_time_unit(_, :native, :millisecond)`); no unit mismatch present on inspection.

