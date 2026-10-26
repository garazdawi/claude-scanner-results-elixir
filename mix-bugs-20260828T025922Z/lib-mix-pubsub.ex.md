# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/pubsub.ex
<!-- sha256: 74a79a793cf8725cacafb9c79fa56a769154786f87ecf44ad52cc7c86a3c996a -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:19:49Z_

## Summary

`Mix.PubSub` is a small internal supervisor-management module exposing `start/0`, `child_spec/1`, and `start_listeners/0`; I found no reachable behavioural bugs — the control flow, error handling, and flush ordering are all consistent with the documented single-caller usage.

## Bugfixes

No findings

## Risks / caveats

- This module is `@moduledoc false` (internal); I treated its exported functions (`start/0`, `child_spec/1`, `start_listeners/0`) as the reviewable surface, since the scanner scope calls out these internal singletons as effectively public to every task.
- Whole-flush-ordering correctness ("deliver each event once, in order") depends on `Mix.PubSub.Subscriber` (a different file, not reviewed here). Within *this* file, `flush/0` is invoked exactly once — on the first successful `start_child` — which is correct; I did not audit the subscriber's accumulator logic.
- I did not exercise a concurrent race between two simultaneous `start_listeners/0` callers as a live test; I reasoned it out (see ruled-out below). In normal Mix execution there is a single caller, so the `already_started` branch is not reached.

### Ruled out

- **`start_listeners/0` does not call `flush/0` on the `{:error, {:already_started, _pid}}` branch (lines 67–68).** Not a bug: `Mix.PubSub.ListenerSupervisor` is *only ever* started from inside `start_listeners/0`, and the sole `{:ok, _pid}` branch that starts it (line 63–65) always flushes immediately afterward. Therefore any state in which the supervisor is already started is a state in which `flush/0` has already run. The top-level `Process.whereis` short-circuit (line 59) is safe for the same reason. Reaching the `already_started` branch at all requires two concurrent callers — no single documented caller can hit it, and even then the winner has already flushed.
- **`raise RuntimeError, "..."` (lines 36, 71–72) passing a string as the second `raise/2` argument.** Verified not a bug: `defexception` generates an `exception/1` binary clause for structs with a `:message` field, so `raise RuntimeError, "msg"` produces a proper exception. Confirmed by running `elixir -e 'try do raise RuntimeError, "failed reason: #{inspect(:boom)}" rescue e -> IO.puts(Exception.message(e)) end'` → prints `failed reason: :boom`.
- **`configured_listeners/0` list concatenation + `Enum.uniq` (lines 91–99).** Both `Application.get_env(:mix, :listeners, [])` and `Keyword.get(config, :listeners, [])` default to `[]`; concatenation and `Enum.uniq` on two lists are total. No missing-clause or shape hazard.
- **`built_in_listeners/0` referencing `IEx.Supervisor` / `IEx.MixListener` when IEx is absent (lines 101–109).** `Process.whereis/1` returns `nil` for an unregistered name without requiring the module to be loaded; the `IEx.MixListener` module reference is only reached (as a child spec, unevaluated) when IEx is actually running. No crash on non-IEx runs.
- **`child_spec/1` return shape vs `@spec Supervisor.child_spec()` (lines 41–54).** The returned map (`id`/`start`/`type`) is a valid `Supervisor.child_spec()`; matches the spec.
- **`start/0` guard `Process.whereis(Mix.PubSub)` (line 25).** The inner supervisor is registered under the name `Mix.PubSub` (opts on line 47), so the guard correctly detects an already-running instance; consistent with the `{:error, {:already_started, _}}` fallback.

