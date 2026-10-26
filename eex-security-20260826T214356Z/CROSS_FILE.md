# Cross-file findings

## Asserted-dependency ledger

No report in this set contains an explicit `* Depends on:` line (the `grep` returned no matches). The dependencies below are the implicit cross-file reliances extracted from prose hedges and delegation chains (Step 2).

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| `lib-eex-smart_engine.ex.md` | `engine.ex` `handle_assign/1` reuses parsed `@name` atom, no data→atom | Confirmed | engine.ex Ruled-out §`handle_assign`/`fetch_assign!`; smart_engine re-verified by reading engine.ex + OTP29 execution. |
| `lib-eex-smart_engine.ex.md` | `engine.ex:198` `String.to_unsafe_atom("arg#{vars_count}")` bounded by counter | Confirmed | engine.ex Ruled-out §`String.to_unsafe_atom` (verified: 2nd same-size template = 0 new atoms). |
| `lib-eex-smart_engine.ex.md` | `engine.ex` `fetch_assign!/2` non-raising, processes value not key-space | Confirmed | engine.ex Ruled-out §`fetch_assign!` (500 evals, 0 atoms, returns nil). |
| `lib-eex-smart_engine.ex.md` | `engine.ex:218-221` marker `/`,`|` rejected via `EEx.SyntaxError` | Confirmed | engine.ex report describes handle_expr; smart_engine executed marker `"|"`→SyntaxError. |
| `lib-eex.ex.md` | `EEx.Compiler` builds the `SyntaxError` snippet/message (raw bytes) | Confirmed | compiler.ex owns `code_snippet` (lines 563/570); eex.ex correctly locates the sink there. |
| `lib-eex.ex.md` | atom creation lives in `EEx.Compiler`/`EEx.Engine`, not this file | Confirmed | compiler.ex Finding `tokenize-atom-exhaustion` + engine.ex atom analysis account for it. |
| `lib-eex.ex.md` | `:parser_options`/`:engine` operator-controlled, handled in compiler | Confirmed | compiler.ex Ruled-out §engine dispatch / parser_options passthrough. |
| `lib-eex-compiler.ex.md` | engine callbacks (`init`/`handle_*`) return valid buffer state | Confirmed | engine.ex + smart_engine.ex verify callback bodies (`check_state!` shape guard). |
| `lib-eex-compiler.ex.md` | template embedded Elixir is trusted-as-code on compile/eval paths | Confirmed | Shared threat-model invariant; eex.ex confirms eval executes / compile+tokenize defer. |
| `lib-eex-engine.ex.md` | template AST arrives already parsed by `Code.string_to_quoted!` in compiler | Confirmed | compiler.ex Ruled-out §`Code.string_to_quoted!` (lines 372/436). |
| `lib-eex-mix.exs.md` | real attack surface lives in sibling source files | Confirmed | Coverage note only; sinks reviewed in compiler/engine/eex reports. |

## Findings

The one genuine cross-file coupling in this folder — `SmartEngine` delegating its entire security posture to `EEx.Engine` — was not left as an assertion: the `smart_engine.ex` re-review explicitly read `engine.ex` and re-executed the atom/marker axes on OTP 29, and `engine.ex`'s own report independently substantiates every reused contract (deterministic `argN` atoms, non-raising `fetch_assign!`, marker rejection). The `eex.ex → compiler.ex` `SyntaxError`-snippet hand-off is consistent across both reports: `eex.ex` correctly places the raw-byte sink in the compiler and both agree it is latent, sink-less defense-in-depth, not a defect. The atom-exhaustion surfaces are cleanly partitioned (`compiler.ex:83` tokenize-classification vs. `engine.ex:198` `argN`) with no double-count or gap. No report rules out a sink that another report shows to be live, and no "safe" verdict rests on an unverified caller-side validation.

No cross-file findings.
