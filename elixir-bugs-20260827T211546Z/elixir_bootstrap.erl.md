# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_bootstrap.erl
<!-- sha256: 531ff40ad218635ddfbd8483da18cec2fd8daf7cc1f4b9277664f104d7f1bb87 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T05:59:03Z_

# elixir_bootstrap.erl — bug-hunting review

## Summary
This is a tiny bootstrap shim exposing a handful of Elixir-macro entry points that delegate to `elixir_module`/`elixir_quote`/`Elixir.Kernel`, and I found no defects reachable through its documented macro API.

## Bugfixes

No findings.

Notes from the pass:

- **Export ↔ `__info__` parity is consistent.** Every macro advertised by `'__info__'(macros)` (`@/1`, `def/1`, `def/2`, `defmacro/1`, `defmacro/2`, `defmacrop/2`, `defmodule/2`, `defp/2`) has a matching `MACRO-` export at the correct arity (user-arity + 1 for the `Caller`), and none is advertised without a backing clause. `'__info__'(functions)` returns `[]`, matching a module that defines only macros. (read `elixir_bootstrap.erl:8-40`)
- **`unless_loaded/3` dispatch is correct.** `'MACRO-@'/2` passes its own name and the received `[Caller, Tree]` unchanged; when `'Elixir.Kernel'` is loaded it `apply`s `'MACRO-@'/2` on it, otherwise returns `nil` — the documented "no-op unless Kernel is defined" behaviour. (read `elixir_bootstrap.erl:14-15,60-64`)
- **No `-spec`s in the file**, so the spec-totality checklist is empty; nothing to reconcile against declared-but-narrower accept sets.

## Risks / caveats
- The review is scoped to this file. `define/4` and `'MACRO-defmodule'/3` rely on the `Caller` being the `{Line, S, Env}` triple with `Env` a map containing both `module` and `line` keys — this is the bootstrap-time `__CALLER__` shape produced by the compiler, not attacker-controlled source text, so a `badmatch`/`badkey` there is a broken-contract-from-an-internal-caller case, explicitly out of scope. If the compiler ever invoked these macros with an env map lacking `line`, `E#{line := Line}` (`:57`) would raise `badkey`, and a `module`-less env would fail the `#{module := Module}` head (`:42`); both depend on `elixir_expand`/parser internals not in this folder slice.

### Ruled out

**Operator / precedence typo**\
Reason: the only boolean expression is `UC or UE` (`:47`), a plain non-short-circuit disjunction whose intent (escape if either side has unquotes) is satisfied correctly for all four truth combinations; no guards with `,`/`;` exist.\
Axes examined: both-false, one-true, both-true for `UC/UE` | Not examined: none.\
Basis: verified

**Missing clause / catch-all**\
Reason: each `MACRO-` head handles exactly the arities `__info__` advertises, and `'MACRO-defmodule'/3` intentionally matches only `[{do, Block}]` because that is the only form the compiler emits for `defmodule`; there is no `case`/`try` in the file that could fall through.\
Axes examined: `defmodule` with `[{do,_}]` body, def/defp/defmacro with and without body arg | Not examined: `defmodule` with a non-`do` option list (compiler-internal, cannot arise from bootstrap sources).\
Basis: verified

**Wrong atom / wrong symbol**\
Reason: atoms (`def`/`defp`/`defmacro`/`defmacrop`, `escape`, `line`, `module`, `elixir_module`, `store_definition`, `'__ENV__'`, `?kernel`) each map to their intended target and the `Kind` atom passed to `define` matches the macro name in every clause.\
Axes examined: each Kind atom vs its clause, `env()` tuple shape | Not examined: none.\
Basis: verified

**State / accumulator threading error**\
Reason: `define/4` has no accumulator loop; `Line` and `E` are threaded once into `E#{line := Line}` and the correct, freshly-bound values are used.\
Axes examined: single-pass data flow through define | Not examined: none.\
Basis: verified

**Off-by-one / wrong arithmetic**\
Reason: no arithmetic in the file (`erlang:unique_integer/0` is an opaque key, not computed).\
Axes examined: all integer/line uses | Not examined: none.\
Basis: verified

**Contract violation**\
Reason: `'MACRO-defmodule'/3` and `define/4` return the documented quoted-AST call-tuple shape `{{'.',[],[Mod,Fun]},[],Args}`; `'MACRO-@'` returns either Kernel's result or `nil` as documented.\
Axes examined: each return shape | Not examined: none.\
Basis: verified

**Crash on stray / unexpected input**\
Reason: all inputs originate from the compiler at bootstrap, not from source text; there is no documented public API here that accepts arbitrary user bytes, so a `function_clause` requires an internal-contract violation excluded by scope.\
Axes examined: caller-tuple shape, macro body option-list shape | Not examined: env map missing `line`/`module` (internal-contract, out of scope).\
Basis: asserted\
Depends on: `elixir_expand`/parser — supplies `__CALLER__` as `{Line, S, Env}` with `Env` a map holding `module` and `line`; if violated the head/update would raise, but only via non-public internal callers.

**Resource leak on error path**\
Reason: the file allocates no files, ETS, processes, or monitors; `write_cache` is a delegated call whose resource ownership lives in `elixir_module`.\
Axes examined: all side-effecting calls | Not examined: none.\
Basis: verified

**Silent corruption**\
Reason: values are passed through structurally (escape/cache-key selection) with no truncation, rounding, or lossy transformation.\
Axes examined: escape-vs-cache branch outputs | Not examined: none.\
Basis: verified

**Infinite loop / hang**\
Reason: no recursion, no `receive`, no loops in the file.\
Axes examined: whole file | Not examined: none.\
Basis: verified

**Quadratic-where-linear / Repeated work**\
Reason: no loops or accumulations; each macro clause is straight-line.\
Axes examined: whole file | Not examined: none.\
Basis: verified

**Dead / unreachable code**\
Reason: every clause is reachable via the corresponding advertised macro; `unless_loaded/3`, `define/4`, `env/0` are each called.\
Axes examined: call sites of each helper | Not examined: none.\
Basis: verified

**Documented behaviour mismatch**\
Reason: the only prose contract is the header comment "Mock out @ to be a no-op unless Kernel is defined"; `'MACRO-@'/2` returns `nil` exactly when `'Elixir.Kernel'` is not loaded and delegates otherwise.\
Axes examined: Kernel-loaded and Kernel-unloaded | Not examined: Kernel loaded but not exporting `'MACRO-@'/2` (would raise `undef`, but Kernel always defines `@` once loaded — build invariant).\
Basis: asserted\
Depends on: `Elixir.Kernel` — once `module_loaded/1` is true it exports `'MACRO-@'/2`; guaranteed by the Kernel source, not verifiable in this file.

**Race condition / shared mutable state**\
Reason: no shared state touched here; any table writes are inside `elixir_module:write_cache/3`, out of file.\
Axes examined: side-effecting delegations | Not examined: none.\
Basis: verified

**Wrong scope / order of operations**\
Reason: `E#{line := Line}` is computed before being handed to `cache_env`, and unquote-detection precedes the escape/cache choice — order matches intent.\
Axes examined: define/4 statement order | Not examined: none.\
Basis: verified

**Encoding / unicode mishandling**\
Reason: no byte/codepoint handling in the file; all values are opaque quoted-AST terms.\
Axes examined: whole file | Not examined: none.\
Basis: verified

**Unclassified**\
Reason: nothing file-specific outside the above classes surfaced.\
Axes examined: full file re-read | Not examined: none.\
Basis: verified

### Did I verify this?

- The `verified` entries above each rest on a full read of the 68-line file with the datatype axes for that class enumerated to `none`; because the module contains no loops, no `receive`, no arithmetic, and no local resource allocation, several classes are vacuously verified (there is no code of that shape to fail).
- Two entries remain `asserted` and I am keeping them so, with explicit `Depends on:` lines, because they hinge on out-of-file behaviour a single-file read cannot cover:
  - **Crash on stray / unexpected input** — depends on `elixir_expand`/parser always supplying a `{Line, S, Env}` caller whose `Env` map carries `module` and `line`. Verification belongs in those modules' reports; from a public-API standpoint these macros are not user-input sinks, so I did not upgrade it.
  - **Documented behaviour mismatch** — depends on `Elixir.Kernel` exporting `'MACRO-@'/2` whenever it is loaded. This is a Kernel build invariant not checkable here; left `asserted`.

