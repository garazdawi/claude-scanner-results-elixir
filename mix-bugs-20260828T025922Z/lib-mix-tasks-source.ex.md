# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/source.ex
<!-- sha256: 846012e9e2a8492c5c649c36e56c2da7e3a6371d2ecb8937cbc845f32fe5fbe1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T05:06:17Z_

Both crashes are confirmed end-to-end through the documented `mix source` entry point. Let me write up the finding.

## Summary

`Mix.Tasks.Source` is a small, single-task file that prints (or opens) the source location of a module/function; its one real defect is that `run/1` only guards against the `:error` return from `IEx.Introspection.decompose/2` and then blindly feeds every other decomposition into `source_location/1`, which crashes with a raw `FunctionClauseError` on shapes `decompose` legitimately produces.

## Bugfixes

#### `CSCAN-source-decompose-malformed-shape-crash`

- **Finding ID:** `CSCAN-source-decompose-malformed-shape-crash`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/mix/tasks/source.ex:43-48`, `Mix.Tasks.Source.run/1`
- **Class:** Missing clause / non-total function
- **Impact:** A user who mistypes the `MODULE.FUN/ARITY` form — e.g. a non-integer arity (`mix source Enum.map/baz`) or an extra call segment (`mix source Enum.foo.bar`) — gets a cryptic `FunctionClauseError` from deep inside `IEx.Introspection.source_location/1`, complete with an internal AST dump and stack trace, instead of the intended clean `Mix.raise("Invalid expression: …")`/`"Could not find source …"`. The error message tells the caller nothing about what to fix.
- **Evidence:** `verified`. In a fresh `mix new` project:
  - `mix source Enum.foo.bar` →
    `** (FunctionClauseError) no function clause matching in IEx.Introspection.source_location/1` with argument `{{{:., …, [{:__aliases__, …, [:Enum]}, :foo]}, …, []}, :bar}`, raised at `lib/mix/tasks/source.ex:48`.
  - `mix source Enum.map/baz` →
    `** (FunctionClauseError) no function clause matching in IEx.Introspection.source_location/1` with argument `{Enum, :map, {:baz, [line: 1], nil}}`, raised at `lib/mix/tasks/source.ex:48`.
  - Control `mix source Enum.map/2` correctly prints `.../enum.ex:1722`.
  Root cause: `decompose/2` can return a 2-tuple whose module element is a non-atom call AST (from `Mod.fun.other`) or a 3-tuple whose arity is a non-integer AST (from `Fun/nonInteger`). Both are non-`:error`, so `run/1` passes them to `source_location/1`, whose three clauses are all guarded (`is_atom`, `is_integer(arity)`) with no catch-all. (Contrast IEx's own `open/1` and `source/1`, which decompose then dispatch through clauses that include an `invalid` catch-all; `source.ex` omits that guard.)
- **Suggested fix:** Before calling `source_location/1`, constrain the accepted shapes, e.g. match `decomposition` against `mod when is_atom(mod)`, `{mod, fun} when is_atom(mod) and is_atom(fun)`, and `{mod, fun, arity} when is_atom(mod) and is_atom(fun) and is_integer(arity)`, and fall through to `Mix.raise("Invalid expression: #{module}")` for anything else — mirroring the existing `:error` handling.

## Risks / caveats

- The two triggering inputs are malformed relative to the documented `MODULE`, `MODULE.FUN`, `MODULE.FUN/ARITY` forms, so severity is Low: the cost is a confusing crash/UX rather than data loss or a security issue. I judged it in scope because the input reaches the defect through the documented `mix source` CLI entry point and the crash message is uninformative, which the review brief explicitly counts as a bug.
- The faulty clause coverage lives in `IEx.Introspection` (outside the reviewed file), but the reviewed file (`source.ex`) is where the incomplete guard/error-handling decision is made; the fix belongs there, since it owns the contract of turning arbitrary user input into a clean error.
- I did not exercise the `--open` path end-to-end (it shells out to `ELIXIR_EDITOR`/`EDITOR`); its return-shape handling (`{:ok, result}`/`{:error, message}`) matches `open_location/2`'s documented contract, so I saw no defect there by inspection.

