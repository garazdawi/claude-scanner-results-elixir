# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/dynamic_supervisor.ex
<!-- sha256: 4d58038be32f32fa31d39791460bb28c847115fe52ea269b80ce4f1313be1646 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:01:39Z_

# Bug-hunting review: `dynamic_supervisor.ex`

## Summary
After enumerating candidate defects across the documented public API (`child_spec/1`, `start_link/1,3`, `start_child/2`, `terminate_child/2`, `which_children/1`, `count_children/1`, `stop/3`, `init/1`, the `init/1` callback, and `use DynamicSupervisor`) and driving the most bug-prone paths with the built Elixir 1.21.0-dev toolchain, I found no genuine, reachable bug — this is a faithful port of the OTP supervisor semantics.

## Bugfixes

No findings.

## Risks / caveats
- I reviewed the whole file but concentrated verification on the public entry points and the state/accumulator-threading and clause-exhaustiveness sites most prone to injected defects (`count_children` reduce, restart-intensity accounting, `max_children` gating, name extraction, restart/terminate lifecycle). The internal `{:start_task, ...}` `handle_call` clause (line 723) is Task.Supervisor plumbing, not part of `DynamicSupervisor`'s documented API, so per scope I did not treat its `Process.get(Task.Supervisor)` assumption as a reportable finding.
- The `sup_flags()` type declares `intensity`/`period`/`strategy`/`max_children`/`extra_arguments` as required keys. `init/2` (line 624) defaults a *missing* `:intensity` to `1` (vs. the documented default of `3` supplied by `DynamicSupervisor.init/1`). This only diverges if a `c:init/1` callback hand-builds a flags map that omits `:intensity`, which violates the `sup_flags` type contract — out of scope, and identical to upstream behavior.

### Ruled out
- **`count_children/1` accumulator order** — the reduce returns `{active, workers, supervisors}` and the reply keyword list maps them correctly. Basis: `verified` — ran with 2 live children → `%{active: 2, workers: 2, supervisors: 0, specs: 2}`; restarting children are excluded from `active` by construction.
- **`map_size(children) < :infinity` gating** — relies on Erlang term ordering (numbers sort before atoms), so `:infinity` yields unbounded capacity. Basis: `verified` — 50 children admitted under the default `:infinity`; a `max_children: 2` supervisor correctly returned `{:error, :max_children}` on the 3rd.
- **`child_spec/1` name `case` (line 212)** — handles atom / `{:global, _}` / `{:via, _, _}`, the full set of GenServer name shapes; `nil` default resolves to the `DynamicSupervisor` atom. Basis: `verified` — atom, global, and default forms produced correct specs.
- **`init/1` callback name `cond` (line 603)** — `is_nil` is checked before `is_atom`, so `nil` (itself an atom) correctly takes the `{self(), mod}` branch. Basis: `asserted` from clause order, corroborated by successful start_link in every test.
- **Restart-intensity shutdown (`add_restart`, line 1038/1052)** — window filter `now <= then + period` and `length(restarts) <= max_restarts` match OTP semantics. Basis: `verified` — a `:permanent` child crashing past `max_restarts: 2` brought the supervisor down with reason `:shutdown`.
- **`maybe_restart_child` clause ordering (lines 989–1014)** — `:permanent` restarts on all reasons; `:transient` skips restart on `:normal`/`:shutdown`; `:temporary` never restarts. Basis: `verified` — a `:transient` child exiting `:normal` was deleted (specs stayed 50, no restart).
- **`terminate_child/2` not-found path** — returns `{:error, :not_found}` for an already-terminated pid. Basis: `verified`.
- **Invalid child-spec / `:ignore` / `extra_arguments` handling** — invalid start returns `{:error, {:invalid_child_spec, _}}`; `extra_arguments` are prepended to child args (`[:extra | args]`); failed starts surface as `{:error, reason}` without adding a spec. Basis: `verified`.
- **`which_children`/`monitor_children`/`count_children` restarting-vs-live tuple shapes** — restarting entries stored as `{:restarting, {mfa,restart,shutdown,type,modules}}` and live as the bare 5-tuple; every consumer's patterns cover both. Basis: `asserted` from exhaustive pattern inspection, consistent with verified `which_children`/`count_children` output.

