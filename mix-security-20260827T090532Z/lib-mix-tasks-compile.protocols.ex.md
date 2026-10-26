# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.protocols.ex
<!-- sha256: 55cbfbe8be5f373d92ce17e9820312f383907fcbbbf678de0e56aff2213b7d27 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.protocols.ex
<!-- sha256: 55cbfbe8be5f373d92ce17e9820312f383907fcbbbf678de0e56aff2213b7d27 -->

_Re-analysed on 2026-08-27_

Full body (13 lines of code, after license header):

```elixir
defmodule Mix.Tasks.Compile.Protocols do
  @moduledoc false
  use Mix.Task

  # TODO: Deprecate me on Elixir v1.23
  def run(_args) do
    :noop
  end
end
```

- **Phase 1 inventory:** The only public entry point is `run/1` (the `Mix.Task` behaviour callback, invokable via `mix compile.protocols`). It binds its single argument to `_args` (discarded) and returns the literal atom `:noop`. Zero sinks of any class are present: no code eval/compile, no atom creation, no `System.cmd`/shell, no `File`/`Path` operations, no `:zip`/`:erl_tar` extraction, no `binary_to_term`, no interpolation, no dynamic dispatch, no network I/O, no crypto, no ETS/persistent_term writes, no `Module.concat`.

- **Phase 2:** Empty — no candidate sinks to trace. `run/1` is total and constant: for every `_args` value it returns `:noop` without inspecting it. No reachable path from the documented API to any dangerous operation.

## Summary

`Mix.Tasks.Compile.Protocols.run/1` is a deprecated no-op that discards its argument and returns `:noop`, exposing no sink of any kind. The one in-file hedge from the first pass — whether `use Mix.Task` injects a data-dependent path — has been verified by reading `Mix.Task.__using__/1` and is resolved: it does not.

## Security

No findings

## Ruled out

- **`use Mix.Task` injecting a runtime sink into `run/1`** — Confidence: high. *Upgraded from hedged assumption in first pass* ("its expansion was treated as trusted infrastructure and not audited here"). I read `/opt/elixir/lib/mix/lib/mix/task.ex:139-148`. The `__using__(_opts)` macro expands to exactly two things: `Enum.each(Mix.Task.supported_attributes(), &Module.register_attribute(__MODULE__, &1, persist: true))` and `@behaviour Mix.Task`. `supported_attributes/0` (line 151) returns the fixed literal list `[:shortdoc, :recursive, :preferred_cli_env, :requirements]`. Both effects are purely compile-time (attribute registration + behaviour declaration); neither injects any code into the body of `run/1` and neither introduces a data-dependent runtime path. The `_opts` argument is ignored, and this call site passes none. Nothing in the expansion reaches an eval/atom/file/shell/network sink at the point of `run/1` invocation.
  - Axes examined: what the macro expands to (attribute registration, behaviour attachment); whether the expansion adds runtime code to `run/1`; whether `supported_attributes/0` is data-dependent (it is a constant literal). | Not examined: none.

- **Residual protocol-consolidation risk** — Confidence: high (scope), not a finding for this file. Historically this task performed protocol consolidation (`Protocol.consolidate` / manifest writes / `File`/`Path` ops); that logic has been removed from this file and now lives in `Mix.Tasks.Compile.Elixir` / `Mix.Compilers`. This is not a hedged assumption about *this* file's behaviour — it is an accurate statement that the code is absent here (verified by full read: the body is a constant `:noop`). Any consolidation risk must be reviewed where that code now lives; it is genuinely out of this file's scope, not an unverified claim about the file under review.
  - Axes examined: presence of any consolidation/file/manifest logic in this file (none — full 13-line body read). | Not examined: none (the referenced logic is not in this file, so there is no in-file axis to try).

## Did I verify this

- The file is a constant function: verified by full read of all 13 lines. No execution needed because there is no data-dependent branch.
- The `use Mix.Task` expansion: verified by reading `Mix.Task.__using__/1` and `Mix.Task.supported_attributes/0` at `/opt/elixir/lib/mix/lib/mix/task.ex:139-153`. Confirmed compile-time-only, no runtime sink.
- No sandbox execution was warranted: there is no input-carrying axis to push through two implementations — `run/1` ignores its argument entirely, so there is no differential behaviour to reproduce.

## Dropped in re-review

None. The first pass reported "No findings"; that conclusion stands, and the two "Risks / caveats" hedges have been converted to verified "Ruled out" entries (one upgraded-then-verified, one confirmed as genuine out-of-file scope) rather than dropped.
