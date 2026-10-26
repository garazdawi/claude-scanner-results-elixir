# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/server.ex
<!-- sha256: fe83b142c94c4da21bfc6295fc5ff2f7f7ec0fd411ba23c6a50f35fda86b4186 -->

_Analysed on 2026-08-27T05:36:15Z_

Confirmed the full picture. The callers pass only trusted, compile-time data:

- `add_module(module, module.__ex_unit__(:config))` — `module` is a compiled test-module atom; `config` is the ExUnit.Case configuration baked in at compile time. Both are the developer's own code.
- `restore_modules/2`, `take_async_modules/1`, `take_sync_modules/0`, `modules_loaded/1` — driven by `ExUnit.Runner`/`ExUnit` internally with integers/booleans and modules the server itself previously handed out (an internal round-trip).

No network bytes, file contents, archive entries, or dependency-supplied data reach any function here. There is no parser, decoder, deserializer, atom-conversion, path, or eval sink in the file. The only shared-mutable-state surface is the `:public` ETS table, which is by-design (OnExitHandler) and reachable only by co-located processes that already have in-VM code execution.

## Summary

`ExUnit.Server` is an internal (`@moduledoc false`) GenServer that queues test modules (async/sync/grouped/parameterized) for the runner and creates the shared `:public` ETS table used by `OnExitHandler`; it contains no code-execution, deserialization, atom-creation, path, or network sinks, and every input reaching it is trusted compile-time data from the developer's own test modules or the internal runner — the security posture is clean.

## Security

No findings

### Ruled out

- **`:public :named_table` ETS table (`server.ex:59`)** — created world-writable so `OnExitHandler` (a different process) can store callbacks. Any co-located process can corrupt it, but that process is already running arbitrary code inside the same BEAM as the test suite, so this is no privilege crossing. Not reachable from network/file/dependency bytes. Out of scope per the folder threat model (co-located user ≈ code-exec).
- **Assertion-style head matches** — `%{waiting: nil, loaded: :done, async_groups: []} = state` and `true = :queue.is_empty(...)` (`take_sync_modules`, line 80–81), the strict `{nil,[module]}`/`{group,_}` reduce in `restore_modules` (line 89–95), and the missing `{:take_async_modules, _}` clause when `waiting` is non-nil. These crash the server on internal-invariant violation, but they are reachable only through the internal runner protocol (single caller, fixed call order), not through any external input, and only self-DoS the already-trusted suite. Not a security defect.
- **`raise "... #{inspect(name)} ..."` (line 35)** — interpolation is via `inspect/1` of a module atom; no injection into a downstream interpreter. Safe.
- **Resource use (`Enum.map(parameterize,…)`, `Enum.uniq`, `take_until/3`)** — all bounded by developer-controlled test config/module counts; no external cost amplifier.

## Risks / caveats

- The `:public` ETS table named `ExUnit.Server` is a genuine shared-mutable-state surface, but its writers/readers live in `on_exit_handler.ex`; the actual read/write/delete sinks (and any integrity assumptions about entries) are out of this file's scope and were flagged for that file's review, not here.
- I treated `name`/`config` from `add_module/2` as trusted because both are compile-time products of `ExUnit.Case`/`use ExUnit.Case`. This holds for normal `mix test`; if some caller ever fed externally-sourced module metadata into `add_module` (e.g. a dynamically-built parameterize list from a file), the `parameterize` fan-out and group `Map.new` would merit a fresh look — no such caller exists in this tree.
- I did not audit the `OnExitHandler`/`ExUnit.Server` cross-process ETS contract end-to-end (whether a hostile entry planted in the table can crash the reader); that is an `on_exit_handler.ex` concern and should be confirmed there.

