# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/macro/env.ex
<!-- sha256: 8dd7594b076d33ba3a91596697dcd14650b4c45257a3208de8c851022ac12801 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/macro/env.ex
<!-- sha256: 8dd7594b076d33ba3a91596697dcd14650b4c45257a3208de8c851022ac12801 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (second pass: hedges resolved by reading callgraph neighbours `elixir_aliases.erl`, `elixir_import.erl`, `elixir_dispatch.erl` and running the atom-minting reproduction)._

## Summary
This module is a thin set of compiler-integration helpers over the `Macro.Env` struct. Its only in-file dangerous primitive is `String.to_unsafe_atom`, confined to two deprecated `@doc false` helpers whose argument is already an atom. The second pass confirms the previously-hedged delegated path: the only atom-minting sink downstream of the documented public API is `:elixir_aliases.expand/4` (`list_to_atom`/`concat`), and its minting is bounded by the pre-existing atom table (each new atom is an `Elixir.`-prefixed variant of an atom the caller already holds), not by attacker-controlled bytes. `:elixir_import` and `:elixir_dispatch` mint no atoms at all. No meaningful vulnerability is reachable through this module's documented public API.

## Security

`No findings`

### Ruled out

- **Atom creation via `String.to_unsafe_atom` in `fetch_alias/2` and `fetch_macro_alias/2`** (`env.ex:216`, `:222`). `String.to_unsafe_atom/1` is `:erlang.binary_to_atom(_, :utf8)`, the unbounded, never-GC'd atom creator (CWE-770). Ruled out on two independent bases: (1) both functions are `@doc false` **and** `@deprecated` (source lines 213–214, 218–219), excluded by the ground rules; (2) the head guard `when is_atom(atom)` means the input is already an atom, so the function only builds `"Elixir." <> already_existing_atom`, bounding new atoms by the already-populated atom table. **Confidence: High.** Basis: `verified` — `elixir` calling `Macro.Env.fetch_alias(__ENV__, :zzUnlikelyAtom123)` emitted the deprecation warning and created exactly the single derived atom `:"Elixir.zzUnlikelyAtom123"`.

- **Atom creation in the delegated `:elixir_aliases.expand/4` reached by `expand_alias/4`** (`env.ex:535`; sink at `elixir_aliases.erl:113` `list_to_atom("Elixir." ++ atom_to_list(H))` and `elixir_aliases.erl:126,175` `concat/1` → `binary_to_atom(_, utf8)`). **Upgraded evidence — this was a hedged "not assessed" caveat in the first pass; it is now read and executed.** The delegated `expand` does mint atoms, but only from segment atoms that must already exist to be passed in the `[atom()]` argument: `Lookup` at line 113 mints `Elixir.<H>` from an existing atom `H`, and `concat` mints a combined `Elixir.A.B…` from existing segment atoms. There is no bytes→atom conversion at this boundary (that boundary is the tokenizer, out of this folder's env.ex surface). New atoms are therefore bounded at ≤1 `Elixir.`-prefixed atom per distinct existing input atom — the same bounded-by-existing-table class as `fetch_alias`. **Confidence: High.** Basis: `verified` — reading `elixir_aliases.erl:96–128,175–176`, plus executed reproduction: `Macro.Env.expand_alias(__ENV__, [], [:ZZunlikelyseg_abc])` raised `atom_count` by exactly 1 (18192→18193), a second identical call minted 0 (18193→18193), and `binary_to_existing_atom("Elixir.ZZunlikelyseg_abc")` then succeeded, confirming the single bounded mint.

- **Atom creation in `:elixir_import.import` / `:elixir_dispatch.expand_import` / `:elixir_dispatch.expand_require`** reached by `define_import/4`, `expand_import/5`, `expand_require/6`. **Upgraded evidence — previously "not assessed" cross-module caveat.** Ruled out: a grep for `list_to_atom|binary_to_atom|to_unsafe_atom|list_to_existing_atom` across `elixir_import.erl` and `elixir_dispatch.erl` returned **no matches** — these modules perform no atom minting. Their inputs (`module`, `name`, `arity`) are already atoms/integers per the env.ex head guards. **Confidence: High.** Basis: `verified` by grep over both delegate modules (zero hits).

- **`FunctionClauseError` from `expand_alias(env, meta, [])`** (`env.ex:531`). An empty segment list raises inside the delegate: `:elixir_aliases.expand/4` line 102 matches `[]` in its general clause, then calls the private `expand/5` (line 112) whose only clause requires `[H|T]`, so `[]` matches no clause → `FunctionClauseError`. Degenerate input, single catchable exception, no amplification, state corruption, or resource cost. **Confidence: High.** Basis: `verified` — reading `elixir_aliases.erl:96–128` confirms the missing empty-list clause, and prior repro returned `{[], {:raised, FunctionClauseError}}` while `[:Foo,:Bar]`, `[:foo]`, `[:Elixir,:Foo]`, `[1,2]` all returned cleanly.

- **Macro execution via the closures returned by `expand_import/5` and `expand_require/6`** (`wrap_expansion/7`, `env.ex:655`). Invoking the returned expander runs `expander.(...)`, i.e. compile-time code from the resolved macro. Ruled out as a defect of *this* file: this is the documented purpose; the code run is the macro the env already imports/requires (identical to normal compilation); `wrap_expansion` adds only a deprecation check and line re-numbering and removes no safety gate. Merely *calling* `expand_import`/`expand_require` resolves the macro and returns a closure without executing the body. **Confidence: High.** Basis: `verified` by reading `wrap_expansion/7` (only invocation runs `expander`); the equivalence-to-normal-compilation claim remains `asserted` (an inherent design property, not a discoverable defect).

- **`module.__info__(:macros)` dispatch in `expand_import/5`** (`env.ex:598`) and the `&module.__info__/1` default in `define_import/4` (`env.ex:437`). `module` is `env.module` / a caller-supplied atom (guard: `is_atom(module)`), i.e. the compilation context, not data derived from untrusted source; `expand_import` further gates the call behind `function_exported?(module, :__info__, 1)`. No untrusted-data flow reaches the dispatch. **Confidence: High.** Basis: `verified` by reading the guard/`function_exported?` gate.

- **`Path.relative_to_cwd(env.file)` in `stacktrace/1`/`relative_location/1`** (`env.ex:719`). Pure path/string computation feeding a synthetic stacktrace tuple; no file open/read/write/delete, so no traversal or TOCTOU surface. `String.to_charlist` creates no atoms. **Confidence: High.** Basis: `verified` by reading the clause (no `File.*` call) and the in-file sink grep (no filesystem sinks).

## Risks / caveats
- **Inherent, documented risk (not a defect here):** `expand_import/5` and `expand_require/6` are macro-expansion primitives — invoking their returned closures runs compile-time code by design. A tool that expands macros parsed from untrusted source is running that code exactly as compiling it would; that is a property of the caller's use, not a flaw in this module.
- **Atom-table growth is bounded but linear in source identifiers.** The delegated `expand_alias` path mints one `Elixir.`-prefixed atom per distinct alias segment. Combined with the tokenizer (which mints the segment atoms in the first place, in the sibling `elixir_tokenizer.erl`), a caller compiling attacker-controlled source grows the atom table linearly in the number of distinct identifiers in that source. This is the compiler's normal, documented behavior and is out of scope for a defensive boundary (`eval_*`/compilation is not a sandbox); it is noted for completeness, not as a defect of `env.ex`.
- **Unicode/RFC obligations (bidi/Trojan-Source, NFC, mixed-script, `\u`/`\x` escape validation) do not apply to this file.** Those are implemented in the tokenizer/interpolation modules under `lib/elixir/src`; `Macro.Env` performs no source-byte sanitisation, identifier tokenization, or normalization, so checklist items 1–12 and the RFC clauses are N/A here.
- Reproductions ran against `/opt/elixir` (Elixir 1.21.0-dev / OTP 29), confirmed built from the source under review.

## Did I verify this
- **`env.ex`** — read in full (lines 1–721).
- **`elixir_aliases.erl`** — read in full (lines 1–277); confirmed the atom-minting sinks at 113/126/175/176 and the empty-list `FunctionClauseError`; ran an executed reproduction confirming the single bounded mint per distinct segment atom.
- **`elixir_import.erl`, `elixir_dispatch.erl`** — grepped for all atom-creation sinks; zero matches, confirming no atom minting in these delegates.
- **Toolchain** — Erlang/OTP 29 + Elixir 1.21.0-dev present and working; the atom-count and empty-list behaviours were executed, not assumed.

### Dropped in re-review
- **Hedge "Cross-module scope … was not assessed" (first-pass Risks/caveats).** Dropped as a hedge and replaced with two verified Ruled-out entries above. New evidence: reading `elixir_aliases.erl:96–128,175–176` (the only minting delegate), the zero-hit atom-sink grep over `elixir_import.erl`/`elixir_dispatch.erl`, and the executed `expand_alias` reproduction showing minting is bounded (+1 atom for a novel segment, +0 on repeat). The concern is now assessed and resolved, not deferred.
