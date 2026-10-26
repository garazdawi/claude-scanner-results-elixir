# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/on_exit_handler/supervisor.ex
<!-- sha256: 7b2896663bab0aa86e0420bd3243603bd7e94ffeb4c4152a2fb1a52c852294c6 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:34:57Z_

# Review — `on_exit_handler/supervisor.ex`

- **Summary:** This is a 28-line `@moduledoc false` internal supervisor that captures the `$callers` chain at `start_link` time and re-installs it in the supervisor process; I found no reachable defect.

## Bugfixes

No findings

## Risks / caveats

The entire module is `@moduledoc false` and every function except `start_link/1`/`init/1` is `defp`. `init/1` is a `Supervisor` behaviour callback invoked only by OTP, and `start_link/1` is called only from `ExUnit.OnExitHandler`, not by end users. So there is effectively no *documented public API* surface here; anything a caller could break requires either driving internal ExUnit processes or violating the `Supervisor.start_link` contract, both out of scope.

### Ruled out

- **`get_callers(self())` capturing the wrong process's callers.** `get_callers(self())` is evaluated inside `start_link/1`, which runs in the *calling* process (the test/setup process). It reads that process's `:"$callers"` dictionary and prepends the owner, exactly the intent — the children spawned under the supervisor must inherit the caller chain of whoever started it. Had the `:erlang.get` been moved into `init/1` (which runs in the freshly-spawned supervisor process, whose dictionary is empty), the chain would be lost; but the code reads it in the correct process. Not a bug. (Verified by reading: `self()` and `:erlang.get(:"$callers")` are both in the `start_link`/`get_callers` path, which OTP runs synchronously in the caller before spawning.)

- **`get_callers/1` clause coverage.** `case :erlang.get(:"$callers")` handles three real return shapes: a non-empty list `[_|_]` → `[owner | list]`; an empty list `[]` → falls to `_` → `[owner]` (correct, nothing to preserve); `:undefined` → `_` → `[owner]`. Every value `:erlang.get/1` can return for this key is covered; no `CaseClauseError` is reachable. Not a missing-clause bug.

- **`max_restarts: 1_000_000, max_seconds: 1` restart intensity.** This is a deliberate "effectively unbounded" intensity so that a per-test supervised child crashing repeatedly does not tear down the on-exit supervisor and take out the surrounding test run. It is the documented ExUnit design (`start_supervised` children are expected to be flaky user code), not a converge-failure bug. Not a supervisor-strategy defect.

- **`Supervisor.init(children, ...)` with arbitrary `children`.** `children` is threaded straight from `start_link/1` into `Supervisor.init/2`. Passing a malformed child spec would raise, but that raise originates in `Supervisor.init/2` on caller-supplied input that violates the `Supervisor` child-spec contract — internal-interface misuse, explicitly out of scope, and the raise is the documented behaviour.

- **`put_callers/1` return value / side effect.** `:erlang.put/2` sets the process dictionary and its return value (the previous value) is discarded, which is fine because `init/1`'s meaningful return is the following `Supervisor.init/2` tuple. No error-swallowing or stale-state threading.

