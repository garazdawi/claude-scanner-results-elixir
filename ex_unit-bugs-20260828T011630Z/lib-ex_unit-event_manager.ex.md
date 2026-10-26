# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/event_manager.ex
<!-- sha256: d933728fe03dcec10f106443ec13e47a4bf5af774537c80cf34fe92625a9d006 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:29:31Z_

## Summary

`ExUnit.EventManager` is a small `@moduledoc false` internal fan-out module (one documented function, `start_link/0`); its logic is correct and I found no defects reachable through a documented public API.

## Bugfixes

No findings

## Risks / caveats

- The module is entirely `@moduledoc false` except `start_link/0`; per the review scope, defects reachable only through the undocumented internal API (`notify`, `stop`, `add_handler`, the various `*_started`/`*_finished` casts) are out of scope. `start_link/0` matches its `@spec start_link() :: {:ok, manager}` (returns `{:ok, {sup, event}}`, where `manager :: {pid, pid}`).
- I did not exercise the deprecated GenEvent path against a live suite, only reasoned about the `function_exported?(handler, :handle_call, 2)` discriminator; it correctly distinguishes gen_event handlers (`handle_call/2`) from GenServers (`handle_call/3`), but this is an `asserted` reading of that one branch.

### Ruled out

- **`stop/1` racing a just-died child (`GenServer.stop` → `:noproc` exit).** I verified `GenServer.stop(pid, :normal, :infinity)` on an already-dead pid exits with `{:noproc, …}` (`elixir t.exs` → `caught exit: {:noproc, {GenServer, :stop, [...], ...}}`). In `stop/1`, a child that dies between `DynamicSupervisor.which_children/1` and the `GenServer.stop` call would crash the caller. Excluded as a finding: `stop/1` is undocumented internal API (not reachable through the documented public surface), and it is a narrow shutdown-time race, not a defect on valid documented input.
- **`notify/2` casting to a dead child.** `GenServer.cast` to a dead pid is a silent no-op, so the analogous race in `notify/2` is harmless. Also internal-only.
- **`add_handler/3` GenEvent-vs-GenServer detection.** The `handle_call/2` (gen_event) vs `handle_call/3` (GenServer) arity distinction is sound; no misclassification of an ordinary formatter GenServer. Internal-only regardless.
- **`stop/1` / `notify/2` `which_children` returning `:restarting`.** Children are registered with `restart: :temporary`, so they are never restarted and `which_children` cannot yield a `:restarting` placeholder pid; the `for {_, pid, _, _}` destructuring binds a real pid.

