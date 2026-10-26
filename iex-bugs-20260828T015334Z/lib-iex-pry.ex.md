# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/pry.ex
<!-- sha256: 00514a57df6954a1b846fef2ee7212603f65caa3caf98463db775cb5c4cdf670 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:13:57Z_

## Summary

`IEx.Pry.remove_breaks/1` crashes the `IEx.Pry` GenServer (and thereby loses every breakpoint) instead of returning the documented `{:error, :no_beam_file}` when handed a module that is not loadable.

## Bugfixes

#### `CSCAN-pry-remove-breaks-undef-module-crash`

- **Finding ID** — `CSCAN-pry-remove-breaks-undef-module-crash`
- **Severity** — High
- **Confidence** — high
- **Location** — `pry.ex:337` (`remove_breaks/1`) → `handle_call({:remove_breaks, module}, ...)` at `pry.ex:428`, via `deinstrument_if_instrumented/1` (`pry.ex:445`) and `instrumented?/1` (`pry.ex:649-651`).
- **Class** — Missing clause / non-total function (unguarded `module.__info__/1` on an unavailable module) → contract violation + process/message error.
- **Impact** — A caller who invokes `IEx.Pry.remove_breaks(SomeModule)` with a module atom that cannot be loaded (typo'd name, or a module from an app that isn't loaded) does not get the documented `:ok | {:error, :no_beam_file}`. Instead `instrumented?/1` calls `module.__info__(:attributes)`, which raises `UndefinedFunctionError`. This is *not* rescued, so the `IEx.Pry` GenServer terminates. Because that process owns the public ETS table `IEx.Pry`, the table is destroyed on crash and **every breakpoint set in the session is silently lost** when the supervisor restarts a fresh, empty server. The caller additionally receives an `exit` signal rather than a value of the declared return type.
- **Suggested fix** — Guard the reflective lookup so an unavailable module is treated as "not instrumented / no beam file" rather than crashing. Smallest change: make `instrumented?/1` tolerant, e.g. `Code.ensure_loaded?(module) and module.__info__(:attributes)[:iex_pry] == [true]` (or wrap the `__info__` call in `try/rescue UndefinedFunctionError -> false`). With that, `deinstrument_if_instrumented/1` returns `:ok` for an unavailable module and the server stays alive. (The same reflective call in `keep_instrumented/1` and the `reset_break` handler is exposed to the narrower "instrumented-then-purged" case and benefits from the same guard.)
- **Threat note** — Reachable directly through the documented public API `remove_breaks/1` with an ordinary bad argument (a module name that is not currently loadable); no attacker required — a mistyped module name at the IEx prompt is enough. Not remotely attacker-controlled, but it is a trivial, plausible user input that takes down shared shell state.
- **Evidence** — `verified`. With the `:iex` application started:
  ```
  IEx.Pry.remove_breaks(NonExistentModuleXYZ)
  ```
  produced:
  ```
  [error] GenServer IEx.Pry terminating
  ** (UndefinedFunctionError) function NonExistentModuleXYZ.__info__/1 is undefined
      (iex) lib/iex/pry.ex:650: IEx.Pry.instrumented?/1
      (iex) lib/iex/pry.ex:446: IEx.Pry.deinstrument_if_instrumented/1
      (iex) lib/iex/pry.ex:431: IEx.Pry.handle_call/3
  ...
  caller got exit: {{:undef, ...}, {GenServer, :call, [IEx.Pry, {:remove_breaks, NonExistentModuleXYZ}, :infinity]}}
  pry alive after? #PID<0.102.0>   # restarted — a different pid; prior ETS breakpoints gone
  ```
  The server pid changed (crash + restart) and the caller received an `exit`, not `:ok`/`{:error, :no_beam_file}`.

### Ruled out

- **`whereami/3` line ranges / `line = 0`** — Verified by running `whereami/3` against a real file: the context window is correctly `L-radius .. L+radius`, the gutter numbering (`Enum.with_index(min + 1)`) is right, and the bold line matches. `line = 0` (valid per the `non_neg_integer` spec) returns `{:ok, lines}` with no highlighted row, which is harmless since line 0 does not exist. No defect.
- **Wasted id on the `:break` error path** — When `break/4` is called for a module with no debug info and no existing entry, the `counter` is bumped (`counter + 1`) before the `fetch_elixir_debug_info_with_fa_check` failure returns, leaving a gap in the id sequence. Ids are opaque unique handles and gaps are never surfaced through `breaks/0` or any documented contract, so this is cosmetic, not a correctness defect.
- **`line_range/1` `min(line, :infinity)`** — Relies on Erlang term ordering (all integers sort before atoms), so `min(int, :infinity)` correctly yields the integer; the sentinel is collapsed by the final `if min == :infinity`. Verified reasoning against term-ordering semantics; behaves as intended.
- **Missing `handle_call`/`handle_info` catch-all in the GenServer** — All internal messages originate from this module's own functions with fixed shapes; reaching an unmatched clause requires sending crafted messages to the internal process, which is out of scope, and `trap_exit` EXIT messages are absorbed by the default `handle_info`. Not reachable via the documented API.

## Risks / caveats

- The `remove_breaks/1` crash was reproduced with the `:iex` application started under `Application.ensure_all_started(:iex)`; behaviour inside a live IEx session is identical since the same supervised `IEx.Pry` server owns the ETS table.
- The narrower "module was instrumented, then purged/deleted out from under Pry" scenario would let `breaks/0`, `reset_break/1`, and `reset_break/3` hit the same `instrumented?/1` crash. I did not construct that race (it requires external module purging mid-session), so I report only the directly-reachable `remove_breaks/1` path with high confidence; the suggested guard fixes all four call sites at once.
- No other documented public function (`pry`, `annotate_quoted`, `whereami`, `break*`) was found to have a triggerable correctness defect within its documented contract.

