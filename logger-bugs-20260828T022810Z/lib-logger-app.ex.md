# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/app.ex
<!-- sha256: a8f7712b00614c7a3271f384f05e48b4165c2a2098e4631b5ee8d086b1d8f299 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:36:47Z_

I have a verified finding and confirmed its mechanism. Let me write the report.

## Summary

`Logger.App.start/2` wraps the default-handler removal action in an extra list, so `redo/1`'s comprehension silently drops it, and stopping the `:logger` application leaks the Elixir default handler while failing to restore the original Erlang one.

## Bugfixes

#### `CSCAN-app-remove-handler-list-wrapped`

- **Finding ID:** `CSCAN-app-remove-handler-list-wrapped`
- **Severity:** High
- **Confidence:** high
- **Location:** `elixir/lib/logger/lib/logger/app.ex:154-155` — `add_elixir_handler/1`, the `:ok ->` clause; consumed by `redo/1` (line 166) via the state threaded from `start/2:75` into `stop/1:95`.
- **Class:** Pattern match on the wrong shape (state/action threading)
- **Issue:** `start/2` builds its application state as `[add_elixir_handler(...) | revert]`, where every element must be a `{fun, args}` action tuple that `redo/1` replays on stop via `for {fun, args} <- handlers`. On successful handler installation, `add_elixir_handler/1` returns `[{:remove_handler, [:default]}]` — the action tuple wrapped in a **list**. That list element does not match the `{fun, args}` pattern, so the comprehension in `redo/1` silently filters it out. Consequently, when the `:logger` application is stopped (`Application.stop(:logger)` → `stop/1`), the Elixir `:default` handler is never removed. The subsequent `{:add_handler, [:default, ...]}` revert action then fails with `{:already_exist, :default}` (ignored), so the original Erlang default handler is never restored. The pre-Logger logging state is not reinstated: the Elixir handler leaks and the OTP default handler is lost.
- **Suggested fix:** Return the bare tuple: `:ok -> {:remove_handler, [:default]}` (mirroring `remove_erlang_handler/0`, which returns bare `{fun, args}` tuples inside the list). The error/else branches already returning `[]`/`:ok` are harmlessly filtered, but for consistency the error branch should also return `[]`.
- **Impact / threat:** Stopping and restarting `:logger` — done in test suites, releases, and some config-reload flows — leaves global logging state corrupted: the Elixir handler persists and the original Erlang default handler is gone. On a later restart, `remove_erlang_handler/0` captures the leftover **Elixir** handler as if it were the erlang default and stores it into `revert`, compounding the corruption across cycles. No untrusted-input vector, but it is reached through the documented Application lifecycle API.
- **Evidence:** `verified`.
  - End-to-end, after stop the `:default` handler is still the Elixir handler rather than the restored Erlang default:
    ```
    $ elixir -e '{:ok,boot}=:logger.get_handler_config(:default); IO.inspect(Keyword.keys(boot.filters),label: "BOOT filters"); IO.inspect(elem(boot.formatter,0),label: "BOOT formatter"); Application.stop(:logger); {:ok,aft}=:logger.get_handler_config(:default); IO.inspect(Keyword.keys(aft.filters),label: "AFTER filters"); IO.inspect(elem(aft.formatter,0),label: "AFTER formatter")'
    BOOT filters: [:remote_gl]
    BOOT formatter module: Logger.Formatter
    AFTER filters: [:remote_gl]        # still the Elixir handler; erlang default NOT restored
    AFTER formatter module: Logger.Formatter
    ```
  - Mechanism — `redo`'s comprehension drops the list-wrapped action:
    ```
    $ elixir -e 'state=[[{:remove_handler,[:default]}],{:set_primary_config,[:pc]},{:add_handler,[:default,:m,:c]}]; IO.inspect(for {fun,args} <- state, do: {fun,args})'
    BUGGY redo actions (remove_handler LOST): [set_primary_config: [:pc], add_handler: [:default, :m, :c]]
    ```
    A bare tuple in that position is retained (`[remove_handler: [:default], set_primary_config: [:pc], add_handler: [:default, :m, :c]]`).

### Ruled out

- **`stop/0` vs `stop/1` name clash** — Both exist but at different arities (arity-1 Application callback, arity-0 documented helper). Distinct functions; no dispatch ambiguity. Not a bug.
- **Error-branch return `:ok` instead of `[]` (line 158)** — On `add_handler` failure the branch returns `IO.puts/2`'s `:ok`. Like `[]`, it fails the `{fun, args}` match and is filtered by `redo/1`; since the add failed there is no Elixir handler to remove and the erlang handler is correctly re-added by `revert`. Behaviorally harmless — inconsistent, not a defect.
- **Primary filters not removed on `Supervisor.start_link` error path (line 78)** — On start failure `redo(revert)` restores primary config and the erlang handler but does not remove the two just-added primary filters (`:logger_process_level`, `:logger_translator`). This is a genuine partial-cleanup asymmetry, but it requires `Logger.Backends.Internal` (legacy backends) to fail to start — reachable only via deprecated `:backends` config, not the default path — and matches long-standing upstream behavior rather than a shape defect introduced here. Noted under Risks rather than as a triggerable finding.

## Risks / caveats

- The environment prints a latin1-encoding warning; it does not affect the handler-config assertions above.
- The start-failure cleanup asymmetry noted in "Ruled out" (leftover primary filters when `Supervisor.start_link` fails) is worth a follow-up: it only bites when legacy `:backends` are configured and a backend fails to start, but it does leave two primary filters installed with no code path to remove them until the next successful `stop/1`.
- I did not exercise the legacy-backends path (`Logger.Backends.Internal`) at runtime; findings for that supervision subtree belong to its own files, not `app.ex`.

