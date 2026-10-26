# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex.ex
<!-- sha256: cdc696618f1d653131e20912907f61a4c6722e42c0208422d1f8f97c43811d43 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:59:15Z_

## Summary

The `break!/2` macro escapes its `stops` argument instead of evaluating it, so any non-literal stop count (a variable or expression) is stored as an AST tuple rather than an integer, producing a garbage breakpoint that crashes the target function when it fires.

## Bugfixes

#### `CSCAN-iex-break-stops-macro-escape`
- **Finding ID** — `CSCAN-iex-break-stops-macro-escape`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `iex.ex:764-767`, `defmacro break!/2` (specifically line 766, `unquote(Macro.escape(stops))`)
- **Class** — Macro and compile-time errors (argument not evaluated / over-escaped)
- **Impact** — When a caller passes anything other than a literal for `stops` — e.g. `break! URI.parse/1, count`, `break! URI.parse/1, @default_stops`, or `break! URI.parse/1, 2+3` — the macro stores the *AST* of the expression as the stop count instead of its runtime value. `breaks()` then reports a nonsensical count, and the first time the instrumented function is invoked, `:ets.update_counter/3` is called with a tuple in the integer counter slot and raises `ArgumentError` (badarg), so the pried function blows up instead of pausing. The bug is silent at break-setup time (only a spurious "variable is unused" compiler warning hints at it).
- **Suggested fix** — Change `unquote(Macro.escape(stops))` to `unquote(stops)`. Only `ast` must be escaped (it is inspected structurally by `__break__!`); `stops` is merely forwarded to `IEx.Pry.break!` and must be evaluated at runtime. For literal integers the two forms are identical, so there is no regression on the documented literal usage.
- **Threat note** — Not attacker-reachable; this is a local developer-facing debugging API and a pure correctness defect.
- **Evidence** — `verified`.
  - `IEx.break!(URI.parse/1, 2+3)` followed by `IEx.Pry.breaks()` returned `[{1, URI, {:parse, 1}, {:+, [line: 7, column: 37], [2, 3]}}]` — the stop count is the AST of `2+3`, not `5`.
  - `n = 3; IEx.break!(Foo.bar/1, n)` compiled with `warning: variable "n" is unused`, proving the runtime value of the stops argument is discarded by the macro.
  - Reproducing the fire path: `:ets.update_counter(:t, :ref, {2, -1, -1, -1})` against an entry whose counter slot holds `{:+, [], [2,3]}` returned `{:error, :badarg}`, confirming the instrumented function raises when such a breakpoint is hit.

### Ruled out
- **`__break__!/3` clause dispatch (`Mod.fun/arity`, `Mod.fun(args)`, `... when guard`, fallback)** — drove all four AST shapes; each routes to the correct clause and the catch-all raises `raise_unknown_break_ast!/1`. `Macro.decompose_call/1` + `Macro.expand/2` on a bare alias or non-call falls through the `with` `else` to the same raise. No missing clause.
- **`arity in 0..255` guard (`__break__!/3` line 770)** — 255 is the BEAM's maximum arity, so the upper bound is correct, not off-by-one; out-of-range arities fall to the fallback raise.
- **`color/2` improper list `[ansi | string]` (line 639)** — `IO.ANSI.format/2` accepts `ansidata` with a binary tail; verified this is the standard, intended shape.
- **`configure/1`, `configuration/0`, `width/0`, `inspect_opts/0`, `started?/0`, `pry/0` macro, deprecated `after_spawn`** — thin delegations to `IEx.Config`/`IEx.Pry` with no logic of their own in this file.

## Risks / caveats
- The confirmed failure-on-fire (`:ets.update_counter` badarg) was established by driving the counter mechanism directly rather than by pausing a live pry session, because a real takeover needs an interactive IEx server/group leader; the stored-AST and unused-variable facts, however, were observed directly from the public `break!/2` + `breaks/0` path.
- This file is almost entirely documentation and delegation; the only non-trivial logic is the `__break__!` AST dispatch and the `break!` macro, both of which were exercised.

