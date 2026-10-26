# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/event_manager.ex
<!-- sha256: d933728fe03dcec10f106443ec13e47a4bf5af774537c80cf34fe92625a9d006 -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/event_manager.ex
<!-- sha256: d933728fe03dcec10f106443ec13e47a4bf5af774537c80cf34fe92625a9d006 -->

_Re-reviewed on 2026-08-27_

## Summary

`ExUnit.EventManager` (`@moduledoc false`) is internal ExUnit plumbing: `start_link/0` starts a `:gen_event` process plus a `DynamicSupervisor` of formatter GenServers; `add_handler/3` installs a handler; the `notify/2` fan-out casts suite/test lifecycle events (`suite_started`, `test_finished`, `sigquit`, …) to every child. In this re-review I read the sole caller, `ExUnit.Runner`, to eliminate the first-pass hedges about where `handler` and `opts` come from, and I re-ran the dead-pid `cast` behaviour rather than asserting it. Every input reaching this file is trusted ExUnit/operator configuration or runner-constructed test metadata. **No findings.**

## Security

No findings.

### Ruled out

- **`add_handler/3` module dispatch (`Code.ensure_loaded?` / `function_exported?` / `GenServer.start_link` / `:gen_event.add_handler`)** — Confidence: high (dismissal).
  A code-execution/behaviour-dispatch sink shape: `GenServer.start_link(handler, opts)` (line 45) and `:gen_event.add_handler(event, handler, opts)` (line 41) instantiate an arbitrary module named by `handler`. **Provenance now verified by reading the only caller**, `ExUnit.Runner`:
  - `runner.ex:43` — `EM.add_handler(manager, ExUnit.RunnerStats, opts)`: `handler` is the compile-time-literal internal module `ExUnit.RunnerStats`.
  - `runner.ex:76` — `Enum.each(opts[:formatters], &EM.add_handler(manager, &1, opts))`: `handler` is each element of `opts[:formatters]`, the developer-supplied `:formatters` suite config (default `[ExUnit.CLIFormatter]`).
  - `opts` is the ExUnit suite configuration threaded through `run/1` → `run_with_trap/1` → `configure/1` (`runner.ex:40-76`), never a dependency/archive/network/manifest value.
  No atom is created from external bytes here; both `handler` values are trusted developer/operator config. `Axes examined: handler provenance (both call sites in runner.ex), opts provenance (suite config), dynamic-atom creation (none) | Not examined: none.`

- **`notify/2` fan-out TOCTOU (`which_children/1` then per-child `GenServer.cast/2`)** — Confidence: high (dismissal).
  `notify/2` (lines 87-95) enumerates `DynamicSupervisor.which_children(sup)` then `GenServer.cast(pid, msg)` for each child. Children are `restart: :temporary` (line 47), so a formatter that crashes between the `which_children` snapshot and the `cast` is gone by the time the cast fires. **Re-ran the behaviour** (`elixir castdead2.exs`): started a GenServer, `GenServer.stop(pid, :normal, :infinity)`, confirmed `Process.alive?(pid) == false`, then `GenServer.cast(pid, {:test_finished, :payload})` → `cast_to_dead_result: :ok` and the caller survived. So a formatter crash mid-dispatch cannot raise in or wedge the notifier. `Axes examined: cast-to-dead-pid return value (executed, :ok), child restart mode (:temporary, source line 47), untriggered-by-untrusted-input (msg is runner metadata) | Not examined: none.`

- **`module_started/2` & `module_finished/2` `Map.put(test_module, :__struct__, ExUnit.TestCase)`** — Confidence: high (dismissal).
  Lines 61 and 67 reshape an internal `%ExUnit.TestModule{}` into a legacy `ExUnit.TestCase` struct for the deprecated `:case_started`/`:case_finished` events (kept until v2.0 per the inline TODOs). `ExUnit.TestCase` is a compile-time literal atom (no dynamic atom creation), and the payload is internal test metadata built by `Runner.run_module/5` from `module.__ex_unit__()` (`runner.ex:241`). Backwards-compat shim, not a security issue. `Axes examined: atom literal vs dynamic (literal), payload provenance (runner-constructed) | Not examined: none.`

- **No matching sinks** for the remaining Phase 1 classes — Confidence: high (dismissal).
  No `binary_to_term`/deserialisation, no file/path/archive operations, no OS-command or shell execution, no `Code.eval*`/`compile*`, no `String.to_atom`/`Module.concat` on external data, no regex compilation, no crypto comparison, no `Base`/`URI` round-trip, no EEx, no network, no persistent-term/ETS/app-env writes keyed on external input in this file. `Axes examined: full file scan for each sink class | Not examined: none.`

## Risks / caveats

- **Trust model.** ExUnit's threat model treats formatter modules/opts and all event payloads as trusted developer/operator input constructed by `ExUnit.Runner` from the developer's own suite. This is no longer an assumption: reading `runner.ex` confirmed the only two external-looking parameters (`handler`, `opts` in `add_handler/3`) flow solely from `ExUnit.RunnerStats` (literal) and `opts[:formatters]` (developer config), with `opts` being the suite configuration — no dependency-, archive-, or network-derived value reaches this file.
- **Non-security robustness (not a finding).** `stop/1` (lines 23-30) calls `GenServer.stop(pid, :normal, :infinity)` on each formatter; a formatter that blocks in `terminate/2` would hang suite shutdown indefinitely. This requires the developer's own formatter (trusted code) to misbehave — self-inflicted, no privilege crossing. Availability/robustness note only.

## Did I verify this?

- **`handler`/`opts` provenance** — Verified. Read `ExUnit.Runner` (`runner.ex:40-94`); the only two `EM.add_handler` call sites are line 43 (literal `ExUnit.RunnerStats`) and line 76 (`opts[:formatters]`), both trusted config. The first-pass hedge citing "runner.ex:43 and runner.ex:76" without having read them is now backed by the actual source.
- **`notify/2` dead-pid cast** — Verified by execution. Re-ran `castdead2.exs` (Elixir/OTP in sandbox): `cast_to_dead_result: :ok`, caller survived. Not an assumption.
- **`Map.put __struct__` atom** — Verified. The atom `ExUnit.TestCase` is a source-level literal on lines 61/67; payload is `%ExUnit.TestModule{}` from `runner.ex:241`.

## Cleared hedges from first pass

- *"I did not exhaustively audit every formatter that ships with ExUnit (e.g. CLIFormatter, RunnerStats)"* — This hedge is **resolved, not upgraded**. The security question it guarded was whether an untrusted value could reach `handler`/`opts`; that is answered by the caller (`runner.ex`), which shows both are trusted config. The *internal behaviour* of `CLIFormatter`/`RunnerStats` is genuinely out of scope for `event_manager.ex` (transport only) and those files carry their own reviews (see `[[exunit-event-manager-review]]`); nothing in this file's sink analysis depends on their internals. No axis of this file's dismissals rests on an unread formatter, so no low-confidence finding is warranted.

## Dropped in re-review

None. All prior dismissals are preserved; the two provenance-dependent ones were strengthened from asserted to verified (one by reading `runner.ex`, one by re-running the dead-pid cast). No prior finding existed to drop.
