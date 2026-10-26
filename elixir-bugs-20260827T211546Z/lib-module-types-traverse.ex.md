# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/traverse.ex
<!-- sha256: 1ac275aecf19266ea70840d5f53f6d840ab61503f4c0a816971be470eabede22 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:33:30Z_

## Summary

`Module.Types.Traverse.of_expr/3` fails to traverse the specifier (right) side of bitstring segments, so a local function referenced inside a bitstring size expression (e.g. `<<x::size(helper())>>`) is never registered as a call — which, on the `:traverse` path used by every macro, turns valid code into a fatal `CompileError`.

## Bugfixes

#### `CSCAN-traverse-bitstring-specifier-calls-dropped`
- **Finding ID** — `CSCAN-traverse-bitstring-specifier-calls-dropped`
- **Severity** — Critical
- **Confidence** — high
- **Location** — `lib/elixir/lib/module/types/traverse.ex:28-36`, the `of_expr({:<<>>, _meta, args}, …)` clause (specifically line 30, `{:"::", _meta, [left, _right]}`)
- **Issue** — For each bitstring segment `{:"::", _, [left, right]}`, only `left` is traversed; `right` (the type/size/unit specifier) is discarded via `_right`. But the specifier can contain arbitrary expressions — `size(expr)`, `unit`, and `-`-combined forms — and those expressions may contain local function calls. The full inference path handles this: `Module.Types.Of.specifier_size/4` (`of.ex:692`) explicitly runs `Module.Types.Expr.of_expr(arg, …)` over the size argument. `Traverse` (the "inference disabled" mirror used when `infer_mode` returns `:traverse` — i.e. for **every** `defmacro`/`defmacrop`, and for all defs when `:infer_signatures` is off) drops it.
- **Class** — State / accumulator threading (dropped sub-expression — a documented traversal invariant, "find local calls", silently violated for one AST branch)
- **Impact** — A private function used only inside a bitstring size specifier is treated as never called. In the full analyzer this is invisible; on the `:traverse` path it produces both a spurious `function foo/N is unused` warning and, fatally, `** (CompileError) …: undefined function foo/N in …` — the module fails to compile. Because `defmacro`/`defmacrop` bodies *always* use `:traverse` mode, this breaks perfectly valid, ordinary macro code with no special configuration; the call site is far from the size expression, so the error is baffling to the author.
- **Evidence** — `verified`.
  - Missed call, directly: driving `of_expr` on `<<x::size(compute_size())>>` with a recording `local_handler` yields `calls in bitstring size: []` (no `compute_size/0`), whereas a normal call `sz()` is recorded.
  - Fatal compile failure via a plain macro (default compiler, no config change), `/tmp/macro.exs`:
    ```
    defmodule MacroTest do
      defp sz(), do: 8
      defmacro build(x) do
        bin = <<0::size(sz())>>
        quote do: {unquote(x), unquote(bin)}
      end
    end
    ```
    →
    ```
    warning: function sz/0 is unused
    ** (CompileError) Elixir.MacroTest: undefined function sz/0 in 'MACRO-build'/2
    ```
  - Control (same macro with `sz()` used outside a specifier) compiles and returns `{1, 8}`; and with default `def` (inference on) `<<x::size(sz())>>` compiles fine (`<<255>>`). So the failure is specific to the specifier branch on the `:traverse` path.
- **Suggested fix** — Traverse the specifier's embedded expressions instead of discarding `right`, mirroring `Of.specifier_size`. Do **not** simply call `of_expr(right, …)` — the specifier tree contains atoms like `{:size, _, [expr]}` and `{:unit, _, [n]}` whose heads are not local functions. Walk `right` for `{:-, _, [l, r]}` and pull the argument out of `{:size, _, [arg]}` (and any other size-bearing specifier), calling `of_expr(arg, stack, context)` on those arguments only, e.g.:
  ```elixir
  {:"::", _meta, [left, right]}, context ->
    context = of_expr(left, stack, context)
    of_specifier(right, stack, context)
  ```
  where `of_specifier` recurses over `{:-, _, [l, r]}` and traverses the `arg` of `{:size, _, [arg]}` (leaving bare type atoms like `{:integer, _, nil}` untouched).

## Risks / caveats

- The full-inference (`:infer`) path is unaffected because it uses `Module.Types.Expr`/`Of.bitstring`; the bug is confined to the `:traverse` fallback. That fallback is nonetheless routinely reached — every macro body flows through it — so the Critical rating stands regardless of the `:infer_signatures` setting.
- I verified the traversal reaches every other expanded-AST shape I could construct (case/cond/receive/try/fn/for/with/captures/anon-call/struct/map-update/blocks) without crashes or dropped calls; the bitstring specifier is the only dropped sub-expression I found. Coverage was driven by exercising `of_expr` directly with a recording `local_handler` (`/tmp/t.exs`, `/tmp/t2.exs`), not by exhaustive enumeration of the grammar, so an exotic specifier form beyond `size`/`unit` could share the same root cause.

### Ruled out
- **`of_pattern` struct clause passes `kind = :expr` (line 167) in a pattern context** — `kind` is used only as a label in the `{:unknown_struct, kind, …}` / `{:unknown_struct_field, kind, …}` diagnostic tuples (`of.ex:522,552`); it does not alter control flow or acceptance, so at worst it mislabels an error message. Not a behavioral defect. Basis: `verified` (traced `kind` usage in `of.ex`).
- **`super` clause `Keyword.fetch!(meta, :super)` (line 147) can raise `KeyError`** — reachable only if a `{:super, meta, args}` node lacks the `:super` meta key. That key is inserted by the compiler's own `super` expansion (`local_handler` at `types.ex:329` pattern-matches `{:super, _, [_|_]}` bodies it produced); reaching `of_expr` with a hand-forged super node requires bypassing the documented expanded-AST contract. Basis: `verified` for the entry point (`types.ex` builds super meta), `asserted` that no expanded body lacks it.
- **Missing `of_expr` clause for some expanded shape (`FunctionClauseError`)** — I enumerated the expanded-AST node set (special forms `:{} :%{} :<<>> :% :| := :<- :__block__ :cond :-> :case :fn :try :receive :for :with :& :super`, local calls `{atom, meta, args}`, vars `{atom, meta, ctx}`, and `.`-calls) against the clauses and drove 19 representative forms through `of_expr` with no crash (`/tmp/t.exs`, all `OK`). Captures like `&(&1+1)` don't reach here because they are expanded to `fn` before traversal. Basis: `verified`.

