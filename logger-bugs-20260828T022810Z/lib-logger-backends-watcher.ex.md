# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/watcher.ex
<!-- sha256: 304a165dcc96b4ae5ed5a95f47b167677987523396dae088f08babafc2364d79 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:40:03Z_

## Summary

The file is a small, entirely-internal (`@moduledoc false` / `@doc false`) GenServer wrapping a `:gen_event` handler; I found no real, triggerable bug reachable through any documented entry point or framework-driven callback path.

## Bugfixes

No findings.

### Ruled out

- **`init/1` recursion is an infinite loop** — The `_ -> init({handler, args})` branch (watcher.ex:38-39) re-runs `init` after `:gen_event.delete_handler` returns any non-`:module_not_found` value. I verified against the OTP under review that a *present* handler's delete returns the handler's `terminate` value (not `:module_not_found`) and a *subsequent* delete of the now-absent handler returns `{:error, :module_not_found}`. Command: `elixir getest.exs` with two back-to-back `:gen_event.delete_handler/3` calls → output `first delete (present): :terminated_value` / `second delete (absent): {:error, :module_not_found}`. So the recursion converges in one extra step (delete → recurse → module_not_found → add); it cannot loop because the handler is gone after the first delete. `verified`.

- **`logger_has_backends?/0` empty-list inversion** — `which_handlers(Logger) != [Logger.Backends.Config]` (watcher.ex:72) returns `true` (i.e. "has backends, suppress stderr") when the handler list is `[]`, which logically means *no* backends and would ideally print the stderr fallback. This is genuinely inverted for the empty case, but it is not triggerable through any real path: `Logger.Backends.Config` is a persistent gen_event handler that remains registered for the entire lifetime of the backends subsystem; an empty list only arises during full teardown of the `Logger` gen_event manager, at which point watchers are stopped via `terminate/2` rather than the `{:gen_event_EXIT, ...}` stderr path. Reachability through a documented API is nil; `asserted` on reachability, and the logic is directly readable. Not reported as a live defect.

- **`{:shutdown, term}` treated as abnormal** — The guard `reason in [:normal, :shutdown]` (watcher.ex:45) does not match `{:shutdown, term}`, so such an exit falls to the stderr-printing clause. This is a defensible design choice (only bare `:normal`/`:shutdown` are silenced) rather than a defect, and it produces informative output rather than a crash or wrong value.

- **`format_exit/1` duplicate clauses** — Both clauses (watcher.ex:86-87) call `Exception.format_exit/1`; the first strips the `{:EXIT, reason}` wrapper, the second passes the reason through. This is correct, intentional normalization, not dead code producing a wrong result.

- **Mailbox / catch-all** — `handle_info/2` has a catch-all `handle_info(_msg, state) -> {:noreply, state}` (watcher.ex:66-68), so unexpected messages do not accumulate; no leak.

- **`{:error, :ignore}` fake-exit path** — Sending `{:gen_event_EXIT, handler, :normal}` to self and returning `{:ok, handler}` (watcher.ex:28-29) correctly routes to the `reason in [:normal, :shutdown]` clause; the later `terminate/2` calling `delete_handler` on a never-added handler harmlessly returns `{:error, :module_not_found}`, which is ignored.

## Risks / caveats

- The module is entirely undocumented (`@moduledoc false`, `@doc false` on `start_link/1`); its only externally-driven surfaces are the OTP `GenServer` callbacks (`init/1`, `handle_info/2`, `terminate/2`), which are exercised only by the supervisor/`:gen_event` machinery with well-shaped messages. There is no attacker-controlled/untrusted-data entry point in this file.
- My reproduction confirmed the `:gen_event.delete_handler/3` return semantics that make the `init/1` recursion terminating; the two edge observations (empty-handler-list inversion, `{:shutdown, term}` handling) are reasoned from code plus the folder's cross-file invariant that `Logger.Backends.Config` is always registered, not independently reproduced, because constructing those states requires driving the full backends supervision tree into abnormal teardown.

