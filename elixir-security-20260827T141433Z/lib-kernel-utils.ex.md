# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/utils.ex
<!-- sha256: 735e14078a81dc497fb185892d68351f1d072ee844f71bfbb6a229b229ac3946 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:32:10Z_

I've read the full file and traced its callers. Here is my assessment.

## Summary
`Kernel.Utils` is an `@moduledoc false` compile-time/runtime callback helper for the `destructure`, `defdelegate`, `defstruct`, `defguard`, and `raise` macros; every input it processes is the developer's own source (at compile time) or a developer-chosen runtime value, so no untrusted trust boundary is crossed and I found no security findings.

## Security

`No findings`

### Ruled out

- **Atom creation — `String.to_unsafe_atom/1` (line 391, `unquote_refs_once`).** This maps to `:erlang.binary_to_atom/2` (confirmed at `lib/string.ex:3040`), which is genuine atom-table pressure. But the argument is `"arg" <> Integer.to_string(map_size(acc) + 1)`, i.e. `arg1..argN` where `N` is the count of *distinct variable references in a single `defguard` head* — developer-authored, bounded by source size, and the base names are reused across every compilation (so the set is effectively fixed, not growing). No external byte stream reaches it. Basis: traced the input expression and the `Macro.postwalk` accumulator in-file; not attacker-influenced.
- **Resource consumption — `destructure_nil/1` recursion (line 25-26).** Builds a list of `count` `nil`s. `count` is supplied by the `Kernel.destructure/2` macro as `length(left)` (verified at `lib/kernel.ex:4159`), where `left` is a compile-time list literal of match targets. Bounded by source, not runtime/external data. Calling `Kernel.Utils.destructure/2` directly with a huge integer is misuse of an internal (`@moduledoc false`) function, which is out of scope.
- **Protocol/behaviour dispatch — `raise/1` → `module.exception([])` (line 265-267).** Reached at runtime only when `raise expr` evaluates to an atom (routing confirmed at `lib/kernel.ex:2272-2273`). The atom is whatever the developer's `raise` argument yields; this file faithfully implements documented `raise/1` semantics and introduces no new boundary. An app that converts untrusted input to an atom and raises it owns that decision (and the atom conversion, not this call, is the risk).
- **Struct construction / key injection — generated `body` (lines 182-216).** Both the bootstrapped and non-bootstrapped builders use map-update syntax `%{map | key => val}`, which raises `KeyError` on any key not already in the struct. Unknown-key injection into a struct built from an external `kv` is therefore rejected by construction, not silently accepted. Basis: read both quoted branches.
- **Shared mutable state / TOCTOU — `defstruct/4` ETS access (lines 108-224).** `:ets.member` check-then-`:ets.insert` and the `enforce_keys` read/update all operate on the *currently-compiling module's* own data tables (`:elixir_module.data_tables(module)`), keyed by the module being defined, in the caller's compile process; module-name uniqueness and single-writer discipline are enforced centrally by `elixir_code_server`. Keys are not externally derived; no cross-caller poisoning path.
- **Compile-time execution — `:elixir_expand.expand/3` (line 349, `expand_defguard`).** Expands the `defguard` body expression as part of normal macro expansion. The expression is the developer's own guard source; these callbacks run only during full compilation (never during bare `Code.string_to_quoted`, which does not expand macros), so any compute here is dominated by the fact that compiling untrusted source already executes arbitrary code — explicitly out of the folder's threat model.
- **Template/interpolation.** All `#{...}` uses (`inspect/1`, `Macro.to_string/1`) build human-readable `ArgumentError`/warning message strings raised as exceptions; no value reaches a downstream interpreter (shell, query, EEx, regex). Not a sink.
- **Command exec / file ops / archive / deserialisation / crypto / network / Unicode.** Searched the file: none of these sink shapes are present. The Unicode/tokenizer obligations in the standing checklist belong to `src/elixir_tokenizer*`/`elixir_interpolation.erl`, not this file.

## Risks / caveats
- The ruled-out items are dismissed on structural grounds (bounded, developer-authored, or internal-only inputs) established by reading this file plus the macro call sites in `lib/kernel.ex` and the `String.to_unsafe_atom` definition in `lib/string.ex`; I did not execute a reproduction because each dismissal rests on a boundedness/soundness fact visible in source, not on a behavioural claim. Basis is therefore "traced," not runtime-"verified."
- Scope excludes direct invocation of these `@moduledoc false` functions with contract-violating arguments (e.g. `Kernel.Utils.destructure/2` with an arbitrary large `count`, or `raise/1` with a hand-crafted term) — those are internal-interface misuse, not reachable through the documented macro public API.
- Residual risk is confined to the general "compiling untrusted Elixir runs arbitrary code" property, which is the caller's responsibility per the folder threat model and not something this file can or should defend.

