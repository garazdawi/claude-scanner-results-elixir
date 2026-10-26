# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/port.ex
<!-- sha256: bd87bcf528512ca6333dfe9bec299e77a04f98a3621ea64a6b403e1374c4e54d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:34:26Z_

The test suite confirms the `info/2` normalization is intentional and there's no `doctest Port` (so the non-deterministic moduledoc examples aren't executed). This module is a thin wrapper over `:erlang` primitives, and every behavior I checked matches its documentation.

## Summary
`Port` is a thin, well-behaved wrapper over `:erlang` port primitives; a runtime review of every non-trivial path found no reachable bugs.

## Bugfixes

No findings

### Ruled out

- **`info(port, :registered_name)` normalization** — The special clause converting `[]` → `{:registered_name, []}` and `:undefined` → `nil` was suspected of mishandling one of the three states (no name / has name / closed). Verified by running all three: raw `[]` becomes `{:registered_name, []}`, a registered name passes through as `{:registered_name, name}`, and a closed port's `:undefined` becomes `nil` — matching the `@spec {atom, term} | nil` and the doc "or `nil` if the port is closed". The project's own `port_test.exs` asserts exactly `{:registered_name, []}`. Basis: verified.
- **`info/1` returning wrong shape for closed port** — `nilify(:erlang.port_info(port))` claimed to yield `nil` when closed. Verified: returned `nil` for a closed port, matching `@spec keyword | nil`. Basis: verified.
- **`open/2`, `close/1`, `command/3`, `connect/2`, `list/0`** — Pure inlined delegates to `:erlang.open_port/2`, `:erlang.port_close/1`, `:erlang.port_command/3`, `:erlang.port_connect/2`, `:erlang.ports/0` with matching arities and specs; no transformation logic to get wrong. Basis: asserted (direct one-line delegation, arities confirmed against the file).
- **`demonitor/1,2` defdelegate with default arg** — `defdelegate demonitor(monitor_ref, options \\ []), to: :erlang` generates `demonitor/1` and `demonitor/2`, both of which exist in `:erlang`; the default `[]` matches `:erlang.demonitor/2`'s option semantics. Basis: asserted (arity/option-list match against Erlang API).
- **`monitor/1` spec/`object` shape** — `@spec monitor(port | {name, node} | name) :: reference` delegates to `:erlang.monitor(:port, port)`; the `@doc` description of the `{:DOWN, ref, :port, object, reason}` message and the `{name, node}` object form matches `:erlang.monitor/2`. Basis: asserted.
- **Moduledoc `iex>` examples as doctests** — The non-deterministic `#Port<0.1444>` examples would fail if executed, but there is no `doctest Port` in `lib/elixir/test/elixir/port_test.exs` (grep returned nothing), so they are never run and are not a broken-doctest finding. Basis: verified (grep + test-file inspection).

## Risks / caveats
The file is essentially a documentation-and-delegation shim; the substantive behavior lives in `:erlang` (BEAM) and is out of scope here. My verification ran against the Elixir build under review (note the environment's latin1-encoding warning, which did not affect the port-info results). The only file-local logic is the `info/2`/`nilify` normalization, which I exercised across all reachable states.

