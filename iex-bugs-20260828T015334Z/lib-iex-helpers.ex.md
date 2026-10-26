# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/helpers.ex
<!-- sha256: 3f1e3b66fe0e7a7b7d4d85b6d74a17bfb1d406433f70dd6b407cd80bcaa7dc5b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:05:49Z_

## Summary

`IEx.Helpers.process_info/1` crashes with a `FunctionClauseError` when the inspected process holds a name-based monitor, because `pid_or_port_details/1` has no clause for the `{name, node}` tuple that `:erlang.process_info(pid, :monitors)` returns for such monitors.

## Bugfixes

#### `CSCAN-helpers-process-info-name-monitor-crash`

- **Finding ID** — `CSCAN-helpers-process-info-name-monitor-crash`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `helpers.ex:957` (`print_process_monitors/1`) → `helpers.ex:981-984` (`pid_or_port_details/1`)
- **Class** — Missing clause / non-total function
- **Impact** — Calling the documented helper `process_info/1` on any live process that monitors another process (or port) *by registered name* aborts mid-print with `** (FunctionClauseError) no function clause matching in IEx.Helpers.pid_or_port_details/1`. The overview and links have already been printed, so the output is truncated at the "Monitors" pane, and the error message names an unexported private function that tells the caller nothing about what they did or how to recover. The target process is perfectly valid; the caller cannot avoid the crash except by not inspecting such processes.
- **Suggested fix** — Add a clause to `pid_or_port_details/1` handling the `{name, node}` shape, e.g. `defp pid_or_port_details({name, _node}) when is_atom(name), do: to_process_details(name)` (or simply `inspect/1` it), placed before the existing `is_atom` clause. Note `to_process_details/1` also needs a `nil` clause if `whereis` misses, or use `inspect` for robustness.
- **Threat note** — Not attacker-reachable in a meaningful sense; it is a pure local correctness bug on valid input. `process_info/1` is a documented public helper (`@doc since: "1.19.0"`, listed in the module docs) and monitoring by name (`Process.monitor(name)` / `Process.monitor({name, node})`) is ordinary OTP practice, so this is readily hit inspecting normal GenServers/supervisors.
- **Evidence** — `verified`.
  - Monitor shape: `elixir /tmp/t.exs` printed `monitors info: {:monitors, [process: {:my_named_agent, :nonode@nohost}]}` — the item is a `{name, node}` tuple, matching none of the `is_pid`/`is_atom`/`is_port`/`is_reference` clauses.
  - End-to-end via the public API: `IEx.Helpers.process_info(p)` on a process that had run `Process.monitor(:my_named_agent)` printed the Overview pane, then `## Monitors`, then:
    ```
    RAISED: no function clause matching in IEx.Helpers.pid_or_port_details/1
    ```
  - A name monitor of a *missing* name (`/tmp/t3.exs`) also yields the tuple form `{:nope_not_registered, :nonode@nohost}`, confirming name monitors never arrive as bare atoms.

### Ruled out

- **`to_process_details(name)` (atom clause) → `Process.whereis` returns `nil` → `to_process_details(nil)` FunctionClauseError** — reachable only if `pid_or_port_details/1` is ever called with a *bare atom*. `verified` that name-based monitors (registered or missing) always surface as `{name, node}` tuples, and process links are only pids/ports; so the `is_atom` clause is effectively dead for both call sites (`print_process_links`/`print_process_monitors`). No public path delivers a bare atom here, so no live defect — but note it is a latent trap that the suggested fix above should also cover.
- **`nl/2` `Enum.zip_with` catch-all absence (`helpers.ex:1756-1760`)** — the fn only matches `{:ok, {:module,_}}`, `{:ok, {:error,_}}`, `{:error, {:erpc,_}}`. `:erpc.multicall` can also return `{:throw,_}`/`{:exit,_}`/`{:error,{:exception,_,_}}`, which would raise. But those only occur if the RPC'd `:code.load_binary/3` itself throws/exits, which it does not (it returns `{:module,_}`/`{:error,_}`); node-level failures (down, timeout, no connection) all come back as `{:error,{:erpc,_}}`, which *is* handled. Not triggerable through the documented behaviour of `load_binary`, so ruled out rather than reported.
- **`print_percentage/3` division by zero (`helpers.ex:753`)** — would need `:atom_limit`/`:ets_limit`/`:port_limit`/`:process_limit` to be `0`, which the VM never reports; not reachable.

## Risks / caveats

- I focused on paths reachable from the documented public helpers. Doctests embedded in `@doc` (e.g. `t(Enum)` at `helpers.ex:416-427`, the `pid/port/ref` examples) were not executed as a suite because `IEx.Helpers` doctests depend on runtime state (history, live nodes, editor env) and third-party module type output; the `pid`/`port`/`ref` construction examples were spot-reasoned as correct against `:erlang.list_to_*` formats but not each run individually.
- The verified crash was reproduced against the `/opt/elixir` toolchain present in the sandbox; the `latin1` locale warning is unrelated to the finding (the `FunctionClauseError` is purely structural).
- `recompile/0`, `r/1`, and `c/2` paths depend on Mix/ParallelCompiler return shapes that vary by project state and were reviewed by reasoning only, not exercised end-to-end.

