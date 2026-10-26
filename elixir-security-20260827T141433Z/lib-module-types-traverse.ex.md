# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/traverse.ex
<!-- sha256: 1ac275aecf19266ea70840d5f53f6d840ab61503f4c0a816971be470eabede22 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:12:30Z_

The expander rejects `%mod{...}` (non-atom module) *before* traverse — so `of_struct` always receives a compile-time atom in practice; the missing `is_atom` guard in the expr clause is compensated upstream. That closes the last in-scope crash path.

I have enough to conclude. Summary of what I established by running against the toolchain built from this source:

- **Reachability confirmed**: `of_expr` runs during normal `mix compile`/`Code.compile_string` on every macro body (`infer_mode/2` → `:traverse` for any kind ∉ `[:def,:defp]`) and on all bodies when `infer_signatures: false` (a documented `Code` compiler option) — trust boundary is compiling untrusted source (e.g. a dependency).
- **No crash on valid source**: diverse constructs (case/cond/try/receive/for/with/fn/captures/struct create+update+pattern/binaries/cons) all compiled through `:traverse` (`:compiled_ok`); the literal clause is complete because `Macro.escape` refuses non-AST types and maps every escapable type (incl. non-binary bitstrings) to a handled form; the expander rejects non-atom struct modules pre-traverse.
- **No dangerous sinks**: no atom creation, no eval of external data (`local_handler` is internal compiler state), no file/network/command/deserialization/crypto/Unicode/template handling; recursion depth is bounded by term depth already built by the parser (depth 20000 compiles end-to-end).

# Security review — `Module.Types.Traverse` (`traverse.ex`)

## Summary
This file is a pure, terminating post-expansion AST walker used by the type checker in `:traverse` mode; it creates no atoms, evaluates no external data, and touches no file/network/command/crypto/Unicode sinks, so I found no security-relevant defect reachable through the compiler's public entry points.

## Security

No findings

### Ruled out

- **Non-exhaustive `of_expr/3` → `FunctionClauseError` (compiler DoS).** `of_expr` has no catch-all, so an unhandled node would crash the checker. *Ruled out (verified):* a module exercising case/cond/try (rescue+catch+after)/receive (with `after`)/for (with `into`)/with (with `else`)/`fn`/`&Mod.fun/arity`/`&local/arity`/struct create+update+pattern/bitstrings/cons compiled cleanly through `:traverse` (`RESULT: :compiled_ok`). The literal clause (line 10) is complete: `Macro.escape/1` **refuses** references, ports and closures (`ArgumentError: cannot escape #Reference…`), and maps every escapable type to a form `of_expr` already handles — a non-binary bitstring `<<1::1>>` escapes to `{:<<>>, …}` (line 28), maps/tuples to `{:%{}}`/`{:{}}` (line 56), remote captures to `&Mod.fun/arity` (line 134). Any residual unhandled shape requires a hand-crafted malformed AST, which only a macro can emit — and authoring that macro already means arbitrary code execution at the victim's build time, so a checker crash is no escalation (and violates `of_expr`'s implicit "well-formed expanded AST" contract).
- **Missing `is_atom(module)` guard on the struct *expr* clauses (lines 38/44) vs. the pattern clause (line 165).** *Ruled out (verified):* `%mod{...}` / `%mod{s | ...}` with a variable module is rejected by the expander (`expected struct name to be a compile time atom or alias, got: mod`) before traverse runs, so `of_struct` → `Module.Types.Of.struct_info/5` always receives a compile-time atom.
- **Unbounded recursion / stack blowup on deeply nested source (resource consumption).** *Ruled out (verified) as traverse-specific:* depth-20000 nested expressions compile end-to-end (parse→expand→traverse) in well under a second; `of_expr`/`of_pattern` recurse exactly to the term's nesting depth, which the parser and expander already recursed through first, so this module is never the weakest link. Very large depths (~100k) merely slow the whole pipeline — a general compiler property, not a defect introduced here.
- **Atom-table exhaustion (CWE-770).** *Ruled out (code inspection):* no `String.to_atom`/`binary_to_atom`/`Module.concat`; function/variable names arrive already-atomized from the AST (guarded by `is_atom(name)`), and arity is computed with `length/1`. No external bytes become atoms.
- **Code execution via `stack.local_handler.(…)` (line 205).** *Ruled out:* the handler is built internally in `Module.Types` (`&local_handler(&1,&2,&3,&4, finder)`, `types.ex:230`), not supplied by any caller through a public API of this module; `fun`/`arity`/`meta` are used only as lookup keys into the module's own local-signature table, with no dispatch to an externally named module or function.
- **Quadratic work in `of_struct/6` (`Enum.any?` over `info` per pair, lines 192-197).** *Ruled out:* `info` is the target struct's field list (bounded by the struct definition, not by attacker input), and `pairs` is bounded by source length, so cost is at most O(fields × pairs) with a small constant `fields` — no input-controlled blowup.

## Risks / caveats
- The completeness argument for `of_expr` rests on three externally-verified guarantees rather than an exhaustive enumeration of every expanded form: `Macro.escape/1`'s refusal of non-AST terms, the expander's normalization/rejection of malformed constructs (e.g. non-atom struct names, `super` outside overridable context) before traverse, and observed compilation of a broad construct set. A future change that let a raw non-AST term reach a traversed body, or an expander relaxation, could expose the no-catch-all `of_expr` as a checker crash; today that path is not reachable from valid source.
- I did not fabricate a live crash from untrusted **non-macro** source, because none exists on the paths I exercised — the residual crash surface requires a macro (hence compile-time code execution), which the threat model treats as an actor that already has code execution, i.e. out of scope.
- Scope note: this module is `@moduledoc false`; findings were judged only as reachable through the documented compiler entry points (`Code.compile_string/2`, `mix compile`, and the documented `:infer_signatures` compiler option), not through direct calls to `of_expr/3` with hand-built `stack`/`context`.

