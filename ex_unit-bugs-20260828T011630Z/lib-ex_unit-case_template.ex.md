# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/case_template.ex
<!-- sha256: aa6732badc6576e758d7a0f9f1d210878bfde5047cd0d695366c7da3bb98611d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:25:43Z_

## Summary

`case_template.ex` is a small, correct macro-only module; both documented entry points (`use ExUnit.CaseTemplate` and the `using/2` macro, with and without a variable argument) behave exactly as their `@doc` examples describe.

## Bugfixes

No findings

## Risks / caveats

- The file is 162 lines and entirely compile-time macro code with no runtime data flow, no I/O, no process/resource management, and no external-input parsing. The threat-model classes about capture devices, monitors, ETS, timeouts, and counters do not apply to this file — they live in its sibling modules.
- I verified the source under review is byte-identical to the ExUnit shipped with the built toolchain (`diff … → IDENTICAL`), so the behavioral runs below exercise exactly this code.

### Ruled out

- **`using/2` with an argument (`using options do … end`) mis-binding or dropping the option** — the moduledoc/`using/2` doctest example depends on `unquote(options)[:import_helpers]` seeing the second arg of `use MyCase`. *Verified*: `defmodule MyCase … using options do quote do if unquote(options)[:import_helpers] do def truth(), do: 42 end end end end` then `use MyCase, async: true, import_helpers: true` — the test `assert truth() == 42` passed (`Result: 3 passed`). The `parent`/`result` combination into `{:__block__, [], [parent, result]}` correctly injects both the `use ExUnit.Case` proxy and the user block.
- **Default-var `using do … end` form producing a broken `__using__(_ = opts)`** — *Verified*: a `DefCase` using the no-argument form injected `def hello(), do: :world` and the consuming test passed. The `var \\ quote(do: _)` default yields a valid always-matching pattern that still binds `opts`.
- **Plain `use ExUnit.CaseTemplate` (no `using`) losing setup/context** — *Verified*: `PlainCase` with only a `setup` returning `{:ok, plain: true}` propagated context to `PlainTest` (`assert ctx.plain` passed), confirming the default `__using__`/`__proxy__` path wires `setup_all`/`setup` to `module.__ex_unit__/2` correctly.
- **Hygiene/`__MODULE__` capture in the nested `defmacro __using__`** — `unquote(__MODULE__)` resolves to `ExUnit.CaseTemplate` while the un-unquoted `__MODULE__` resolves to the template module (`MyCase`); the passing multi-module run above exercises both resolutions with distinct modules, so no leakage occurs.
- **`opts` double-evaluation / side effects** — `opts` is compile-time AST (the keyword list passed to `use`), embedded into a quote via `ExUnit.Case.__keys__(unquote(opts))`; there is no runtime expression to evaluate twice. Not a defect.

