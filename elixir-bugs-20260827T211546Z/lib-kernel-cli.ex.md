# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/cli.ex
<!-- sha256: 97588a7eb41e99b073d433cf55ca4106b113126c856bb237a7c4afd379e6f909 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:25:44Z_

## Summary

`Kernel.CLI` (byte-identical to shipped Elixir 1.21.0-dev) is the compiler/CLI boot front-end; after tracing every documented public entry point (`parse_argv/1`, `process_commands/1`, `format_error/3`, `run/1`, `rpc_eval/1`) and its private helpers, I found no genuine reachable defect.

## Bugfixes

No findings

## Risks / caveats

- The module is `@moduledoc false`; its "public" surface is the `@doc`-annotated functions labelled *"Made public for testing"* (`parse_argv/1`, `process_commands/1`) plus `run/1`, `main/1`, `format_error/3`, `rpc_eval/1`. I scoped the review to those and the private clauses they reach. The `+iex`/`+elixirc`/erlang-flag `parse_argv` clauses are only reachable when the boot script injects the mode marker, but they are exercised through the same public `parse_argv/1`, so I treated them as in-scope.
- Behavioural claims were checked against the toolchain built from this source (`Elixir 1.21.0-dev`, `Erlang/OTP 29`). The `System.halt`-terminating paths (`halt_standalone`, the `-v` combined-option path, `run/1`) were reasoned rather than executed to completion, since running them terminates the VM; their logic is straightforward and I did not find a defect in it.

### Ruled out

- **`process_command({:rpc_eval, …})` calls a non-existent `List.to_unsafe_atom/1`** — Ruled out. `List.to_unsafe_atom/1` exists and is exported in this version (`lib/elixir/lib/list.ex:1059-1060`, with its own doctest). Basis: verified (grep of `list.ex`).
- **`append_hostname/1` crashes when the node is not distributed (`:net_kernel.nodename/0` missing or returning a non-list)** — Ruled out. `net_kernel:nodename/0` is exported (`erlang:function_exported(net_kernel, nodename, 0)` → `true`); when it yields no `@`, `:string.find/2` returns the atom `nomatch`, the `[_|_] = suffix <-` clause fails, and the `else _ -> node` branch returns the node unchanged. No crash. Basis: verified (`net_kernel.nodename/0` export) + asserted for the `with`/`else` fall-through.
- **`--rpc-eval` with a missing expression argument raises instead of erroring** — Ruled out. The short clause `parse_argv([~c"--rpc-eval" | _], config)` appends `"--rpc-eval : wrong number of arguments"` to `errors` and returns cleanly. Basis: verified (parse produced `["--rpc-eval : wrong number of arguments"]`).
- **`--dbg <backend>` mis-handled outside `:iex` mode** — Ruled out. In `:iex` mode an unknown backend records the `Unknown dbg backend` error; in default mode `--dbg` correctly falls through to the file/argv fallback (`{:file, ~c"--dbg"}`, remaining argv `[~c"weird"]`), matching documented CLI semantics that `--dbg` is IEx-only. Basis: verified (both parses observed).
- **`parse_argv` loses trailing arguments after `-e`/a script file** — Ruled out. With a prior `:eval` command the fallback returns `{config, [h | t]}` leaving all remaining args as script argv; the first bare arg otherwise becomes `{:file, h}` with the rest returned as argv. Both observed correct. Basis: verified.
- **Multi-`when` guard in `exec_fun`'s exit catch mis-evaluates (`tuple_size(reason)` on an atom)** — Ruled out. Chained `when` clauses are OR-composed and a guard sub-expression that errors (e.g. `tuple_size(:normal)`) simply yields false for that alternative rather than raising, so `:normal`/`:shutdown`/`{:shutdown, _}` are all matched correctly. Basis: asserted (standard Erlang guard semantics; the surrounding shutdown/DOWN handshake sends then exits, and the parent flushes the monitor).
- **`format_error/3` / `run/1` reference missing functions (`Code.loaded?/1`)** — Ruled out. `Code.loaded?/1` is defined (`code.ex:2238`) and `format_error/3` produces correct iodata for a real (blamed) `Protocol.UndefinedError` stacktrace without raising. Basis: verified (grep + executed `format_error/3`).
- **`process_commands/1` reverses `config.errors` in the wrong order / drops errors** — Ruled out. `Enum.reverse(config.errors, errors)` restores parse-time errors (accumulated head-first) to source order and prepends them to command-execution errors, which is the intended ordering. Basis: asserted (data-flow trace; parse-time error accumulation is head-prepended at lines 297, 363).

