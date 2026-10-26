# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/expr.ex
<!-- sha256: e2f9b066f71a5242bf797cdd1691cfe856d2f4ba5bbcec867ddb6945c3055942 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:57:37Z_

## Summary

`Module.Types.Expr` is an internal (`@moduledoc false`) set-theoretic type-inference pass that walks already-parsed/expanded quoted AST to compute types and emit diagnostics; it contains no dangerous sinks and, on review plus reproduction, yields no meaningful security findings.

## Security

`No findings`

This file is the compiler's static type-checker for expressions. Its two exported functions (`of_expr/5`, `format_diagnostic/1`) are compiler internals with no documented public API. I traced every external call and confirmed the module performs only *type-level* reasoning: it never evaluates, applies, spawns, or invokes anything derived from source, never converts source bytes to atoms, and never touches the filesystem, network, OS, crypto, or global mutable state. It is exercised on all compiled source (verified below), so a malicious dependency's source does flow through it — but no sink is present for that data to reach.

### Ruled out

- **Terminal/control-character injection via diagnostics** (`format_diagnostic/1`, lines 1065–1249). The `:badmap`/`:badtimeout`/`:badkey`/etc. messages interpolate source expressions and types via `expr_to_string`, `to_quoted_string`, and `inspect`. *Basis: verified.* Compiled a module whose `receive … after` timeout was the literal `"\e[31mINJECTED\a\e[0m"` and captured the raw diagnostic bytes with `Code.with_diagnostics`: the ESC (0x1b) and BEL (0x07) were rendered as the escaped two-character sequences `\e`/`\a` (`:binary.match(msg, <<0x1b>>) == :nomatch`, same for `<<0x07>>`). Elixir's AST/inspect rendering escapes control characters, so no raw terminal control sequence reaches the console.
- **Arbitrary code execution / atom-table exhaustion.** *Basis: verified (full-file scan).* Read the entire file and grepped for the checklist sinks (`*_to_atom`, `binary_to_term`, `Code.eval/compile`, `Module.concat/create`, `System.cmd/shell`, `:os.cmd`, `Path.*`, `File.*`, `EEx`, `Base.decode`, `URI.*`, `:crypto`, `Regex.compile`/`~r`, `apply`, `spawn`, `send`, `:erlang.*`): the only hit is `Apply.remote_apply(info, Collectable, :into, [type], …)` (line 879), which computes the *return type* of `Collectable.into/1` from a type descriptor — it does not call the function. No atom is ever created from source-derived text (`atom([atom])` wraps atoms already produced by the parser).
- **Compile-time DoS / super-linear blowup in the reverse-arrow cache** (`case`/`cond` handling, lines 289–508; `cache_result`/`cache_arrows`, 958–985). *Basis: verified.* Compiled N-deep nested `case` expressions in subject position (the reverse-arrow-heavy path): compile time was linear (n=8→1.8 ms, 12→2.2, 16→2.6, 20→3.2), confirming the `reverse_arrows` cache prevents re-expansion. Nested `cond` was flat. Chained `=` matches and 5000-deep nested tuples showed only mild ~quadratic growth (2000 `=` → 223 ms; 5000-deep tuple → 1.76 s) that is not disproportionate, is attributable to the shared `Descr`/`Pattern` type algebra rather than this file, and is reachable only by compiling source that would execute at build time regardless.
- **`Keyword.fetch!(meta, :version)` / `Map.fetch!(context.reverse_arrows, …)` crashes** (lines 386–387, 744, 965–971). *Basis: reasoned from the compiler contract.* The `:version` meta and `reverse_arrows` map are populated by the expansion pass, not expressible in surface syntax, so a missing key requires internal-interface misuse (out of scope) and would be a clean build failure, not a security event.

## Risks / caveats

- **Scope of the absence claim.** The "no dangerous sinks" verdict covers *this file only*. The type-level operations it delegates to — `Apply.remote/remote_apply`, `Of.struct_info/modules/closed_map`, `Pattern.of_head/of_match`, and the `Descr` union/intersection algebra — live in sibling modules that were not reviewed here; a sink or a genuinely exponential type-algebra blowup would surface there, not in `expr.ex`.
- **DoS residual.** The measured quadratic behavior on extreme nested inputs (thousands of levels) is real but impractical as an attack and, in the compile-from-dependency threat model, is dominated by the fact that compiling such a dependency already runs its macros. I did not fuzz for a pathological small-input/large-cost combination beyond the shapes tested (nested case/cond/tuple, match chains).
- **Diagnostic rendering** was verified to escape control characters for string-literal and integer paths; I did not separately exhaust every diagnostic clause (e.g. atom-keyed `:badkey` rendering via `to_quoted_string` of an `atom([:"\e…"])` type), though all paths route through the same escaping `Macro`/`inspect` machinery.

