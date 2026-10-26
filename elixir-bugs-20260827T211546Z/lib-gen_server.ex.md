# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/gen_server.ex
<!-- sha256: 16860a5b7429a669bdca3add7a55d09de0cec04d8fa35aa0d37d8e943b73cfe2 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:10:55Z_

## Summary

`gen_server.ex` is the Elixir `GenServer` client wrapper plus behaviour/`use` macro; after inventorying every public entry point and checking the documented contracts, I found no genuine reachable bug.

## Bugfixes

No findings

## Risks / caveats

- Coverage: I reviewed every exported function (`start_link/3`, `start/3`, `stop/3`, `call/3`, `cast/2`, `abcast/3`, `multi_call/4`, `reply/2`, `whereis/1`) and both macros (`__using__/1`, `__before_compile__/1`) plus the injected default callbacks. The behaviour is a thin, well-worn wrapper over `:gen`/`:gen_server`/`:proc_lib`; the meaningful logic is argument dispatch and error-tuple shaping, which I exercised in the sandbox.
- The `iex>`-prefixed examples in the `## Debugging with the :sys module` moduledoc section (lines 470–518) are formatted like doctests but reference an undefined `Stack` module and show environment-specific PIDs and `*DBG*` trace lines that are not valid expected-result syntax. They are **not** a live finding here because Elixir's own test suite does not run `doctest GenServer` against this moduledoc (these blocks are illustrative and match upstream); if a downstream consumer added `doctest GenServer`, they would fail. I did not have the test harness wired to prove non-execution beyond it matching upstream behaviour, so I flag it as a caveat rather than assert it.

### Ruled out

- **`whereis(nil)` / `whereis(:unregistered)` crashing** — `nil` and any unregistered atom flow through the `is_atom/1` clause to `Process.whereis/1`, which returns `nil`, matching the `@spec` and `@doc` ("`nil` otherwise"). Basis: verified (`whereis(nil): nil`, `whereis(atom): nil`).
- **`whereis({name, node})` remote/local dispatch** — local-node tuple resolves via `Process.whereis/1` (returns `nil` when absent), remote tuple is returned verbatim per the documented `{name, node}` return. Clause order (`local == node()` before the generic `is_atom(node)` clause) is correct. Basis: verified (`whereis local tuple: nil`, `whereis remote tuple: {:some_name, :remote@host}`).
- **`cast/2` to missing/global/via targets** — all catch `_, _ -> :ok` and return `:ok` unconditionally, matching the "always returns `:ok`" contract; clause order places `{:global,_}`/`{:via,_,_}`/`{name,node}` before the generic atom/pid clause. Basis: verified (`cast to missing atom: :ok`, `cast global missing: :ok`).
- **`abcast/3` return value** — discards per-node sends and returns `:abcast` as documented/spec'd. Basis: verified (`abcast: :abcast`).
- **`call/3` timeout guard precedence** — `(is_integer(timeout) and timeout >= 0) or timeout == :infinity`: `and` binds tighter than `or`, so a negative or non-integer timeout falls through the guard and raises `FunctionClauseError`, which is contract misuse (documented as non-negative or `:infinity`), not a reachable bug on valid input. Basis: verified (`call neg timeout: FunctionClauseError`).
- **`call/3` result/exit handling** — `try/catch(:exit)/else {:ok, res}` correctly unwraps `:gen.call/4`'s `{:ok, res}` and rewraps exits with the MFA context; no other return shape is produced by `:gen.call`. Basis: asserted against `:gen.call/4`'s fixed `{:ok, Reply}` contract.
- **`do_start/4` `:name` dispatch** — `{nil, opts}` is matched before the generic `is_atom(atom)` clause, so `nil` is not mis-sent as `{:local, nil}`; `{:global,_}`/`{:via,module,_}` (with `is_atom(via_module)`) are handled, and anything else raises the documented `ArgumentError`. Basis: asserted from clause ordering.
- **`stop/3` self/noproc handling** — `whereis` → `nil` exits `:noproc`, `pid == self()` exits `:calling_self`, otherwise `:proc_lib.stop/3` with exit rewrapping; matches the documented synchronous-stop semantics. Basis: asserted from `:proc_lib.stop/3` contract.
- **Injected default `handle_call`/`handle_cast`/`handle_info`** — the `:erlang.phash2(1, 1)` branch always yields `0` (raise) for call/cast (intentional Dialyzer non-local-return trick), and `handle_info` logs via `format_report/1` whose head matches the emitted `{GenServer, :no_handle_info}` report shape exactly. Basis: asserted, cross-checked report label against `format_report/1` pattern at line 1370.

