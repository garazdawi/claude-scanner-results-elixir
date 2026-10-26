# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/typespec.ex
<!-- sha256: caa6f1e2bcc7ff79122bd8b63076e7d1893139fafde294a0147b9340e6cbf4d1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:34:34Z_

Based on my review and testing, here is the report.

## Summary

The typespec translator is robust across the valid and invalid inputs I exercised; the one genuine defect is that nested type annotations re-evaluate the inner type expression, emitting any side-effecting deprecation warning it contains twice.

## Bugfixes

#### `CSCAN-typespec-nested-ann-double-eval`
- **Finding ID** — `CSCAN-typespec-nested-ann-double-eval`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/kernel/typespec.ex:714-726`, `typespec/4` type-operator (`::`) clause, the `{{:ann_type, _, _}, _state}` branch.
- **Issue** — When the right-hand side of a `::` annotation is itself an annotation (`x :: (y :: t)`), the clause calls `typespec(expr, ...)` once to detect the nested-annotation case, throws away that result (`_state`), then calls `typespec(expr, ...)` a *second* time to obtain `right`. The inner type expression `expr` is therefore fully translated twice. Any warning-emitting type inside it (`string()`, `nonempty_string()`, `char_list()`, `fun/N`, the nested-annotation warning itself) fires on both passes.
- **Class** — Macro and compile-time errors (a code path evaluating its argument more than once → duplicated side effects).
- **Impact** — Duplicate compiler warnings for a single source construct, plus redundant work (`Exception.format_stacktrace/1` is run twice). It is cosmetic — the produced abstract code is correct because the second run restarts from the original `state`, so `used_type_pairs`/`local_vars` accounting is not corrupted — but it clutters compiler output and can double-count a deprecation in tooling that parses warnings. Only reachable through the deprecated nested-annotation path (slated for removal in v2.0).
- **Evidence** — `verified`. Compiling
```elixir
defmodule Nest2 do
  @spec f(x :: (y :: string())) :: :ok
  def f(_), do: :ok
end
```
with the freshly built `elixir` and counting the `string() type use is discouraged` warnings:
```
$ elixir /tmp/nest2.exs 2>&1 | grep -c "string() type use is discouraged"
2
```
The same expression with a non-side-effecting inner type (`x :: (y :: integer)`) prints the nested-annotation warning only once, confirming the duplication comes specifically from re-translating `expr`.
- **Suggested fix** — Reuse the already-computed result instead of re-running the translation. Bind the first call and reuse its `state`/right value:
```elixir
case typespec(expr, vars, caller, state) do
  {{:ann_type, _, _} = right, state} ->
    IO.warn(message, caller)
    {{:ann_type, location(meta), [{:var, location(var_meta), var_name}, right]}, state}

  {right, state} ->
    {{:ann_type, location(meta), [{:var, location(var_meta), var_name}, right]}, state}
end
```
(The two branches then differ only by the `IO.warn`, and `expr` is translated once.)

## Risks / caveats

- The file is `@moduledoc false` and every function is `@doc false`; the only realistically "documented" entry surface is `deftypespec/6` (invoked by `@/1` expansion) and the `Module`-hook functions (`defines_type?/2`, `spec_to_callback/2`), so all findings must be triggerable from ordinary user typespec source. The one reported finding is.
- Several `defines_type?/2`, `spec_to_callback/2`, and doc-storage paths (`store_doc`, `get_doc_info`, `get_doc_meta`) depend on module ETS data tables (`:elixir_module.data_tables/1`) that I could not stand up in isolation without driving a full module compile; I exercised them only indirectly through end-to-end compiles, so a defect confined to those ETS-shape assumptions could be under-covered.

### Ruled out
- **Equal-bound / negative-equal ranges (`1..1`, `-1..-1`) rejected by `validate_range`** — verified they raise `Kernel.TypespecError`, but this is Elixir's long-standing intentional `first < last` contract (the error message states it), not a regression or crash-on-valid-input in the bug sense. Basis: verified.
- **`local_vars` / `used_type_pairs` corruption from the nested-annotation double-eval** — I hypothesized the second `typespec(expr)` would double-count a single-use variable and suppress the "used only once" error; verified it does **not**: `@spec f(x :: (y :: a)) :: integer when a: var` still raises the single-use error, because the re-run restarts from the original `state` (the first call's `_state` is discarded). Basis: verified.
- **`keyword/2`, `as_boolean/0`, `charlist/1`, `record/3` producing "undefined type" instead of arity-specific handling** — verified these fall through to the general clause and yield a correct, specific `type X/N undefined` message; no crash or misleading error. Basis: verified.
- **`+1..+3` (unary-plus range bounds) rejected** — `validate_range` handles `:-` but not `:+`, so a leading `+` on a bound yields the generic range error. Judged not a real-world bug: `+N` integer literals are essentially never written and the resulting error is still coherent. Basis: asserted (reasoned from the `validate_range/3` clauses at lines 1023-1041, which pattern-match only `:op … :-`).
- **`fun/N`, funs with `...`, multi-field maps with `required`/`optional`, parametric types, binary specs (`<<_::8,_::_*4>>`, `<<_::0>>`), `nonempty_list`** — all compiled cleanly in the smoke test; no missing-clause or shape bug found. Basis: verified.

