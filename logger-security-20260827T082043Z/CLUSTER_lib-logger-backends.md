# Cluster cross-file findings

#### `CSCAN-cluster-reinit-orphaned-logger-handler`

- **Finding ID:** `CSCAN-cluster-reinit-orphaned-logger-handler`
- **Severity:** Low
- **Confidence:** medium
- **Files involved:**
  - `lib/logger/backends/internal.ex:139, 147, 151-157, 159` (and the whole module — it has **no** `terminate/2`; file ends at 161)
  - `lib/logger/backends/handler.ex:40-41, 76-77, 88-95`
  - `lib/logger/backends/config.ex:74-76`
  - `lib/logger/backends/supervisor.ex:66`
  - `lib/logger/backends/watcher.ex:78-84`
  - (out-of-cluster context) `lib/logger/app.ex:73, 91`
- **Finding shape:** Other — cleanup/lifecycle responsibility split across files re-opens a precondition (`"safe because the caller removes it"` broken); net effect is a logging availability DoS on attacker-reachable input.
- **Evidence:**
  - `internal.ex:151-157` installs a **process-less global `:logger` handler** in `init/1`: `:ok = :logger.add_handler(Logger, Logger.Backends.Handler, %{level: :all, config: %{counter: counter}, ...})`, with a fresh `counter = :counters.new(1, [:atomics])` (`internal.ex:139`). `Logger.Backends.Handler.adding_handler/1` (`handler.ex:15-17`) starts **no** supervised process, so nothing auto-removes this handler when the adding process dies.
  - `Logger.Backends.Internal` is `use Supervisor` (`internal.ex:7`) with **no `terminate/2`** and no `restart:` override → `restart: :permanent`; `app.ex:73` places it under `Logger.Supervisor` (`one_for_one`, default intensity 3/5). Its own children run `strategy: :rest_for_one` (`internal.ex:159`) with default intensity 3/5, over `Logger.Backends.Supervisor` (`supervisor.ex:66`, `max_restarts: 30, max_seconds: 3`).
  - The **only** code that ever removes this handler is `app.ex:91` (`:logger.remove_handler(Logger)`) — and only on application **stop**. No file in the cluster removes it: `watcher.ex:82`'s `terminate/2` calls `:gen_event.delete_handler(Logger, handler, :ok)`, which removes the *gen_event backend handler* (Config/Console/custom), **not** the `:logger` `Logger` handler.
  - Consequence chain: a legacy backend that crash-loops on hostile logged content exhausts `supervisor.ex:66`'s 30/3s budget → `Backends.Supervisor` exits → `rest_for_one` fails `Internal` past 3/5 → `Internal` exits and `Logger.Supervisor` restarts it → `init/1` re-runs → `:ok = :logger.add_handler(Logger, ...)` now returns `{:error, {:already_exist, Logger}}` (handler was never removed) → **MatchError**, so `init/1` crashes *before* `Supervisor.init` starts any child. Internal can never re-initialize; 3 such crashes in 5s take down `Logger.Supervisor` (the Logger app root).
  - During the window Internal is down but the handler is still installed, `handler.ex:88-95` keeps `:counters.add`-ing the now-orphaned counter while `config.ex:74-76` (the only resetter, now dead) can no longer reset it → `value >= discard` becomes permanently true → `log/2` silently returns `:ok`/`:discard` (`handler.ex:41-43`), dropping all events even before the app-level teardown.
- **What the per-file pass missed:** The `internal.ex` reviewer saw `add_handler` succeed once and concluded "no findings"; a single-file view cannot know the handler is never torn down on a *subtree restart* — establishing that requires reading `watcher.ex`'s `terminate` (it deletes a *different* handler), confirming `config.ex`/`console.ex`/`supervisor.ex` never call `:logger.remove_handler(Logger)`, and noting the sole remover (`app.ex:91`) fires only on app stop — and the silent-discard consequence only appears once you see the counter is shared with the now-dead `config.ex` resetter.
- **Suggested fix:** Make `Logger.Backends.Internal.init/1` idempotent: call `:logger.remove_handler(Logger)` (ignoring its result) immediately before the `add_handler` at `internal.ex:151`, so a restart re-adds cleanly with the fresh counter (and the freshly-started `Config` resets the *same* counter). Optionally add a `terminate/2` to `Logger.Backends.Internal` that removes the handler on shutdown for symmetry with `app.ex:91`. Merely tolerating `{:error, {:already_exist, _}}` is **not** sufficient — the stale handler would keep the old counter while the new `Config` resets a new one, so the handler must be removed and re-added.
