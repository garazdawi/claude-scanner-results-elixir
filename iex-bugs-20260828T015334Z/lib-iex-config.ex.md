# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/config.ex
<!-- sha256: 66b281a4f5f70cd47691179ae2cd5c8c0374529e485bf104c95392a343bae906 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:03:29Z_

## Summary
A single `IEx.configure/1` call with an invalid option runs its validation inside the shared `IEx.Config` agent, so instead of returning a clean error it crashes and restarts the agent, silently discarding every registered `after_spawn` hook.

## Bugfixes

#### `CSCAN-config-invalid-option-crashes-agent`
- **Finding ID** — `CSCAN-config-invalid-option-crashes-agent`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `config.ex:164-166` (`configure/1`) → `config.ex:180-188` (`handle_configure/2`) → `config.ex:210-212` (`validate_option/1`)
- **Class** — Process and message errors (uncaught raise inside an `Agent` callback destroys shared state)
- **Impact** — `validate_option/1` is designed to reject bad configuration with `raise ArgumentError, "invalid configuration ..."`. But `configure/1` runs it via `Agent.update(@agent, __MODULE__, :handle_configure, ...)`, so the raise happens *inside* the singleton `IEx.Config` agent process. The agent terminates and is restarted by its supervisor, which re-runs `handle_init` and creates a fresh ETS table — wiping the `:after_spawn` hook list. The caller does not get the intended `ArgumentError`; it gets an opaque `(exit) exited in: GenServer.call ...`. Any hooks registered through the documented public `IEx.after_spawn/1` (or by IEx internals) are gone, so future shell respawns / `pry` sessions silently skip them. Repeated mistakes also chip away at the supervisor's restart budget.
- **Suggested fix** — Validate in the caller process before touching the agent, e.g. `def configure(options) do Enum.each(options, &validate_option/1); Agent.update(@agent, __MODULE__, :handle_configure, [options]) end` and drop the `Enum.each(options, &validate_option/1)` from `handle_configure/2`. The `ArgumentError` then reaches the caller directly and the agent (and its `after_spawn` table) is never disturbed by invalid input.
- **Threat note** — Not attacker-reachable (local trusted user), but trivially hit by a normal fat-finger: a typo in an option name (`IEx.configure(color: ...)` instead of `colors:`) or a wrong value type (`IEx.configure(width: 1.5)`) — precisely the inputs `validate_option/1` exists to reject — destroys global shell state via the documented `IEx.configure/1` entry point.
- **Evidence** — `verified`. Running (elixir 1.21.0-dev built from this tree):
  ```
  after_spawn before: [#Function<0.111634049 in file:/tmp/cfgtest4.exs>]
  agent pid before: #PID<0.102.0>
  02:02:23.627 [error] GenServer IEx.Config terminating
  ** (ArgumentError) invalid configuration {:bogus, 1}
      (iex 1.21.0-dev) lib/iex/config.ex:211: IEx.Config.validate_option/1
      ...
      (iex 1.21.0-dev) lib/iex/config.ex:181: IEx.Config.handle_configure/2
      (elixir 1.21.0-dev) lib/agent/server.ex:27: Agent.Server.handle_call/3
  configure(bogus) caught kind: :exit
  agent pid after: #PID<0.105.0>
  same agent pid?: false
  after_spawn after: []
  ```
  The registered `after_spawn` hook (via public `IEx.after_spawn/1`) is present before the invalid `IEx.configure(bogus: 1)` and empty after; the agent pid changes, confirming crash + restart; the caller catches `:exit`, not `ArgumentError`. (`width: 1.5` and `parser: {1,2,3}`-shape mismatches reach the same raise path.)

### Ruled out
- **`update_configuration/1` anonymous fn (`config.ex:191-193`) has a guarded clause `when key in @keys` and no catch-all** — could raise `FunctionClauseError` on a non-`@keys` key, but not triggerable: `config` is `configuration()` (already `Keyword.take(@keys)`) merged with `options`, and every key `validate_option/1` accepts is itself a member of `@keys`, so the merged result's keys are always ⊆ `@keys`. Verified `@keys` (lines 11-21) and the validated keys (lines 200-208) are the identical set. Dead defensive guard, not reachable.
- **`merge_option/3` `is_list(new)` guards on `:colors`/`:inspect` falling through to `merge_option(_, _, new) -> new` (line 198)** — would replace instead of merge if `new` weren't a list, but `validate_option` (lines 200-201) already requires `is_list` for both keys and runs first, so a non-list never reaches merge. No divergent branch reachable.
- **`width/0` `min(value, columns)` (lines 42-46)** — behaves correctly: returns the smaller of configured width (default 80) and terminal columns (default 80 on `:io.columns` error). Verified output `width: 80` with no terminal. No off-by-one.
- **`prompt/1` continuation generation (lines 25-34)** — `Enum.split_while` on the prompt charlist splits at first `?(`; both branches produce sensibly-padded continuation prompts (`...(1)>` for `iex(1)>`). No index/width error found.
- **`inspect_opts/0` + `update_syntax_colors/1` (lines 132-148)** — `Keyword.update(opts, :syntax_colors, default, &Keyword.merge(syntax_colors, &1))` correctly lets user-supplied syntax colors win while preserving the injected `:reset`. Verified `inspect_opts` returns a well-formed keyword list. Behaves as intended.
- **`parser/0` using `get_env` rather than `fetch_env!` (lines 71-75)** — intentional per its comment (invocable from a remote node without IEx started); default `{IEx.Evaluator, :parse, []}` is correct.

## Risks / caveats
- The reported defect requires an *invalid* option, which technically violates `IEx.configure/1`'s documented contract — normally out of scope. I include it because `validate_option/1` is explicitly built to detect and report that exact input, so the module's own contract is to fail *cleanly*; the defect is that failing cleanly is defeated by running validation in the shared agent, with a real cross-cutting consequence (loss of `after_spawn` hooks registered through the documented public `IEx.after_spawn/1`). Downstream triage should weigh this framing.
- This mirrors the structure of the shipping Elixir source (verified against the 1.21.0-dev build in the sandbox); it is not a local deviation, so it may be considered accepted upstream behavior even though the state-loss consequence is real.
- The module is `@moduledoc false`; I treated its functions as public via their re-exports/callers in `IEx`/`IEx.Helpers` (notably `IEx.configure/1` and `IEx.after_spawn/1`). No `@spec`s are present, so no spec/impl mismatches were assessable.

