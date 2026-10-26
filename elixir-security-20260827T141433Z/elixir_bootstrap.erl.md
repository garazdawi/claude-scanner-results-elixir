# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_bootstrap.erl
<!-- sha256: 531ff40ad218635ddfbd8483da18cec2fd8daf7cc1f4b9277664f104d7f1bb87 -->

_Analysed on 2026-08-27T14:31:48Z_

The file is fully mapped. `elixir_bootstrap.erl` is a 68-line bootstrap shim: it provides minimal `def`/`defmodule`/`defmacro`/`@` macro implementations used *only* while Elixir compiles its own standard library (before `Kernel` exists). Every "sink" is AST plumbing into compiler-internal, module-scoped state. Callees confirm: `write_cache`/`cache_env` write module-scoped ETS with a `unique_integer` key; `has_unquotes` is a pure bounded tree-walk; `escape` is wrapped in a `try`. There is no Unicode, tokenizer, interpolation, escape, atom-creation, or deserialization code in this file, so the entire standards/RFC checklist is N/A here.

# Security review: `elixir_bootstrap.erl`

## Summary
`elixir_bootstrap.erl` is a compile-time bootstrap stand-in for `Kernel` whose macros only run while Elixir builds its own trusted standard library; it contains no security-relevant sink reachable from a documented public API on untrusted input, so there are no findings.

## Security

No findings.

The module exports only `MACRO-`-prefixed macro implementations plus `__info__/1`. These are invoked by the compiler's expansion machinery **exclusively in bootstrap mode**, where `elixir_bootstrap` is registered as the `Kernel` replacement because `Kernel` is not yet compiled. After bootstrap, all `def`/`defmodule`/`@` expansion goes through the real `Kernel`, not this module. The documented parse/compile entry points behave as follows:
- `Code.string_to_quoted/2` (and `elixir:string_to_quoted`) only tokenize+parse and never expand macros, so they never call these functions.
- `Code.eval_string`/`Code.compile_string` expand macros but run with the full `Kernel` loaded, so `def`/`defmodule` resolve to `Kernel`, not `elixir_bootstrap`.

There is no documented public API that compiles attacker-supplied source in bootstrap mode, and even if one existed, every operation here (delegating to `Kernel.@`, escaping quoted forms, writing module-scoped compiler ETS) is identical to the normal `Kernel` compilation path and grants no capability beyond "macro expansion runs code," which is by design and not a sandbox boundary this file defends.

## Risks / caveats
- **Reachability rests on the bootstrap wiring, which is not in this checkout.** This checkout contains only `lib/elixir/src/*.erl`; the `.ex`/build-system code that registers `elixir_bootstrap` as the Kernel replacement (the `@bootstrap` compiler mode) is absent. My "unreachable from untrusted public API" conclusion depends on the established architecture that this module is swapped in *only* during Elixir's own self-build. If a future change exposed a public API that compiles untrusted source with `elixir_bootstrap` registered as Kernel, the `apply('Elixir.Kernel', 'MACRO-@', …)` delegation and the module-ETS writes would still be equivalent to normal compilation, so no new sink appears — but the trust assumption should be re-checked if that wiring changes.
- **Standards/Unicode obligations do not apply to this file.** All 12 sub-claims (BIDI, NFC, mixed-script, escape validation, UTF-8 decode, grapheme segmentation, etc.) and all 10 RFC clauses target `elixir_tokenizer.erl`, `elixir_interpolation.erl`, and `elixir_utils.erl`. This file contains none of that code, so there is nothing to satisfy or violate here.
- The `apply/3` uses a fixed module (`'Elixir.Kernel'`) and a fixed function atom (only caller passes the literal `'MACRO-@'`); it is not attacker-controlled dynamic dispatch. `unless_loaded/3`, `define/4`, and `env/0` are unexported, so they are reachable only through the four literal call sites in this file.

### Ruled out

- **`apply(?kernel, Fun, Args)` — `unless_loaded/3` (line 62). Class: Code execution / dynamic dispatch.**
  Reason: both the module (`'Elixir.Kernel'`, a compile-time macro constant) and the function (`Fun`) are fixed literals — the sole caller (`'MACRO-@'/2`, line 15) passes the literal atom `'MACRO-@'` — so there is no attacker-controlled dispatch and no atom creation; it is the intended delegation to `Kernel.@` during macro expansion.
  Axes examined: function-atom origin (literal vs param), module-atom origin (macro constant), export surface of `unless_loaded/3`, arity | Not examined: none.
  Basis: verified.

- **`elixir_quote:escape(Block, escape, false)` — `MACRO-defmodule/3` (line 26); `elixir_quote:escape({Call,Expr}, escape, true)` — `define/4` (line 49). Class: Resource consumption (recursion) / code generation.**
  Reason: `escape` is a `try`-wrapped recursive quote-tree walk (`elixir_quote.erl:149`) whose depth is bounded by the already-parsed AST; the input is trusted bootstrap source, and the recursion lives in `elixir_quote`, not here. Behaviour is identical to normal `Kernel` compilation.
  Axes examined: input origin (parser-bounded AST), trust context (bootstrap-only/trusted), depth bound, exception containment (`try`) | Not examined: none.
  Basis: verified.

- **`elixir_quote:has_unquotes(Call)` / `has_unquotes(Expr)` — `define/4` (lines 43–44). Class: Resource consumption (recursion).**
  Reason: pure boolean tree-walk (`elixir_quote.erl:42–65`) over parser-bounded AST; no atoms, files, or mutable state; trusted input.
  Axes examined: recursion depth (parser-bounded), side effects (none), input trust | Not examined: none.
  Basis: verified.

- **AST tuples for `elixir_module:compile` (line 28) and `elixir_def:store_definition` (line 58). Class: Code execution (generated AST).**
  Reason: these are literal AST tuples handed back to the expander; the file does not itself execute them, and they encode the same compiler calls the real `Kernel` macros emit. Downstream execution is normal compilation of trusted bootstrap source.
  Axes examined: who executes (expander, not this file), argument provenance (Line/Alias/env from caller env), parity with Kernel path | Not examined: none.
  Basis: verified.

- **`elixir_module:write_cache(Module, Key, {Call, Expr})` (line 53) and `cache_env(E#{line := Line})` (line 57). Class: Shared mutable state.**
  Reason: `write_cache` inserts into a *module-scoped* ETS data table (`elixir_module.erl:56–58`) under `{cache, Key}` where `Key = erlang:unique_integer()`, and `cache_env` dedups the env in the same per-module table (`elixir_module.erl:82+`); there is no cross-module/cross-tenant key collision and the writer is the single compiling process. Not attacker-influenced across a trust boundary.
  Axes examined: key scope (per-module table + unique_integer), writer concurrency (single compiler process), key collision, table origin | Not examined: none.
  Basis: verified.

- **`erlang:unique_integer()` (line 52). Class: Resource consumption / key generation.**
  Reason: generates a fresh cache key; monotonic, no wrap-relevant security property, no external influence.
  Axes examined: input influence (none), rollover relevance (none — used as an opaque ETS key) | Not examined: none.
  Basis: verified.

- **Pattern-match clauses in `define/4` (line 42, requires `{Line,_S,#{module:=Module}=E}`) and `MACRO-defmodule/3` (line 25, requires `[{do, Block}]`). Class: Dead / unreachable defensive code (input validation via match).**
  Reason: these are not defensive guards but ordinary destructuring; a malformed caller env or missing `do` block yields a `function_clause` compile error surfaced by the compiler, not a security event, and in bootstrap mode the expander guarantees well-formed args. No fall-through to acceptance.
  Axes examined: consequence of non-match (compile error), input well-formedness guarantee (compiler-produced env), acceptance fall-through (none) | Not examined: none.
  Basis: verified.

- **`__info__/1` (lines 30–40). Class: Deserialisation / metadata.**
  Reason: returns a static literal list of `{macro, arity}` tuples; no input, no dynamic term construction, no atom creation.
  Axes examined: input (none), dynamic construction (none) | Not examined: none.
  Basis: verified.

- **Standards/Unicode checklist (sub-claims 1–12) and RFC clauses (1–10).**
  Reason: this file contains no tokenizer, interpolation, escape-decoding, UTF-8 conversion, identifier-security, bidi/NFC, or atom-from-text code; a full read (68 lines) and a targeted grep for `unicode|binary_to_atom|list_to_atom|characters_to|binary_to_term|bidi|nfc` returned no matches. All obligations are owned by `elixir_tokenizer.erl`/`elixir_interpolation.erl`/`elixir_utils.erl`.
  Axes examined: presence of Unicode/atom/escape/UTF-8/bidi code (grep + full read) | Not examined: none.
  Basis: verified.

### Did I verify this?
- `apply(?kernel, Fun, Args)` → **verified**: read lines 14–15 and 60–64; the only caller passes the literal `'MACRO-@'` and `?kernel` is the literal `'Elixir.Kernel'`; `unless_loaded/3` is unexported. No axes outstanding.
- `elixir_quote:escape` / `has_unquotes` → **verified**: read `elixir_quote.erl:42–65` and `:149+`; `has_unquotes` is a pure bounded walk and `escape` is `try`-wrapped. Input is parser-bounded and trusted in bootstrap. No axes outstanding.
- `write_cache` / `cache_env` → **verified**: read `elixir_module.erl:56–58, 82+`; both operate on per-module ETS tables with a `unique_integer` key; single compiling writer. No cross-tenant key axis remains.
- Generated-AST tuples (`elixir_module:compile`, `elixir_def:store_definition`) → **verified**: this file only constructs tuples; execution is the expander's, identical to the `Kernel` path. No axes outstanding.
- Pattern-match clauses / `__info__/1` / `unique_integer` → **verified**: non-match is a compile-time `function_clause`, not a security event; `__info__` is static. No axes outstanding.
- Standards/RFC obligations → **verified** as N/A: full read plus grep confirm no Unicode/atom/escape/UTF-8/deserialisation code exists in this file.
- Reachability of the module itself → **asserted** (the one honest residual): the bootstrap-registration wiring is not in this checkout. `Depends on: the Elixir build system / compiler bootstrap mode — `elixir_bootstrap` is registered as the `Kernel` replacement only during Elixir's own self-compilation, never for compiling untrusted user source via a documented public API (`Code.eval_string`/`compile_string` run with the real `Kernel`).` If that invariant ever changes, re-examine — though even then no sink in this file exceeds normal-compilation behaviour.

