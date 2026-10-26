# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/task/supervisor.ex
<!-- sha256: 46f79e40f789d15ed5dc857d2b215c7c016510b691478d7e0699c6866db7eaf4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:56:56Z_

## Summary
The file is a thin, well-behaved wrapper over `DynamicSupervisor`/`Task.Supervised`; the only genuine defect is that the streaming functions reject a `:shutdown` value (`:infinity`) that their own published option type declares valid and that the non-stream `async/*` functions accept.

## Bugfixes

#### `CSCAN-supervisor-async-stream-infinity-shutdown`
- **Finding ID** — `CSCAN-supervisor-async-stream-infinity-shutdown`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/task/supervisor.ex:633` (`build_stream/5`), against the `async_stream_option` type at `lib/task/supervisor.ex:88`
- **Issue** — `build_stream/5` validates `shutdown` with `(is_integer(shutdown) and shutdown >= 0) or shutdown == :brutal_kill` and raises `ArgumentError` otherwise. But the documented option type for all four `async_stream`/`async_stream_nolink` functions is `@type async_stream_option :: ... | {:shutdown, Supervisor.shutdown()}`, and `Supervisor.shutdown()` is `timeout() | :brutal_kill`, i.e. it includes `:infinity`. So `:infinity` is a type-valid, documented value that is rejected at runtime. The sibling non-stream functions (`async/3,5`, `async_nolink/3,5`, whose `async_opts` type also permits `timeout()`) accept `:infinity` and pass it straight through, so the two documented surfaces disagree on the same option.
- **Class** — Contract violation (`@type`/`@spec` disagrees with implementation)
- **Impact** — A caller who relies on the published `Supervisor.shutdown()` option type and writes `shutdown: :infinity` for a streamed run gets an `ArgumentError` at runtime, even though the same value works for `async/*` and is what the type promises. Dialyzer, consuming the declared type, will not flag the call, so the failure only appears at runtime. The raised message is at least informative, keeping this low severity.
- **Evidence** — `verified`.
  - `Task.Supervisor.async_stream(sup, [1,2,3], fn x -> x*2 end, shutdown: :infinity) |> Enum.to_list()` → `RAISED: :shutdown must be either a positive integer or :brutal_kill`.
  - `Task.Supervisor.async(sup, fn -> 42 end, shutdown: :infinity)` then `Task.await` → `async infinity: 42` (accepted).
  - Type check: `Supervisor.shutdown` resolves to `union([timeout(), :brutal_kill])`, and `timeout() = :infinity | non_neg_integer()`.
- **Suggested fix** — Either allow `:infinity` in the guard (`... or shutdown == :infinity`) to match the type and the `async/*` behaviour, or narrow the `async_stream_option` typespec's shutdown value to `{:shutdown, :brutal_kill | non_neg_integer()}` so the contract matches what the code enforces.

## Risks / caveats
- I reviewed only `lib/task/supervisor.ex`; the actual task lifecycle, ordering, timeout, and cleanup logic lives in `Task.Supervised` (`stream/7`, `validate_stream_options/1`) and `DynamicSupervisor`, which are out of scope here — a bug in stream cleanup or option validation there would not be visible from this file.
- Whether the `:infinity` rejection is "wrong" depends on intent: an infinite shutdown timeout for streamed tasks is arguably degenerate, so the maintainers may prefer to fix the *type* rather than the *code*. Either way the current state is an unresolved contract mismatch.

### Ruled out
- **`build_stream/5` message says "positive integer" but accepts `0`** — the guard is `shutdown >= 0`, so `shutdown: 0` is accepted while the error text says "positive integer". Verified `shutdown: 0` returns `[ok: 2, ok: 4, ok: 6]`. This is harmless: `0` is a meaningful (immediate) shutdown value and accepting it is correct; only the error *message* wording is imprecise, and that message is only shown for genuinely invalid values. Basis: `verified`.
- **`child_spec/1` `case` handles only atom / `{:global, _}` / `{:via, _, _}` names** — any other `:name` shape would `CaseClauseError`. Ruled out: those three are exactly the documented `GenServer` name-registration forms (`GenServer` docs), and the default is the atom `Task.Supervisor`; reaching the gap requires passing a `:name` that violates the documented name contract. Basis: `asserted` (name forms enumerated against the documented `GenServer` name-registration set).
- **`start_child/5` guard omits `is_atom(module)`** — unlike `async_stream/6` it does not guard the module argument. Ruled out: passing a non-atom module violates the `@spec` (`module`) contract, i.e. caller misuse, which is explicitly out of scope. Basis: `asserted`.
- **`restart || :temporary` / `shutdown || 5000` in `start_link/1` treating `0` as falsy** — checked: in Elixir only `nil`/`false` are falsy, so `0 || 5000 == 0`; `shutdown: 0` survives. No truthiness bug. Basis: `verified` (Elixir truthiness semantics; `0 || 5000` evaluates to `0`).
- **`start_child_maybe_temporary/6` uses `make_ref()` where `async/6` uses `Task.__alias__(pid)`** — the `start_child` path has no awaiter, so an inert `make_ref` (rather than a real reply alias) is intentional; the task's reply send simply goes nowhere. `start_child` verified to return `{:ok, pid}` and run its side effect. Basis: `verified`.

