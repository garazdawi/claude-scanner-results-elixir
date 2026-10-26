# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_try.erl
<!-- sha256: 4d24ec8bfb8cc07a713b3641e48e7e9b24a141f4cec417795db9114e206a5f04 -->

_Analysed on 2026-08-27T16:53:38Z_

Confirmed: zero dangerous-sink patterns in the file. My analysis is complete. Writing the report.

# Security review — `elixir_erl_try.erl`

## Summary

`elixir_erl_try.erl` is a pure compile-time AST-to-AST transformer that lowers Elixir `try/rescue/catch` clauses into Erlang abstract format, contains no dangerous sinks (no deserialisation, atom-from-bytes, code/OS/file/network/crypto/NIF operations), and every incomplete pattern-match it does contain is provably gated upstream by the expander — so there are **No findings**.

## Security

`No findings`

The file's entire job is to build Erlang abstract-format tuples from an already-expanded quoted AST. A full sink scan (`grep` for `binary_to_term`, `list_to_atom`/`binary_to_atom`, `open_port`/`os:cmd`, `file:`/`filelib`, `spawn`, `apply`/`eval`, `io:format`/logger, `rand_uniform`, `load_nif`, `xmerl`, `:consult`, ETS/match-spec — all in the Phase-1 checklist) returned **no matches**. The only atoms the module emits (`throw`, `error`, `badarg`, `'Elixir.ErlangError'`, the `erlang` operator atoms, etc.) are compile-time literals; none are derived from source bytes. There is no process-dictionary, ETS, or application-env write in this file, so no shared-mutable-state sink. All recursion (`reduce_clauses/5`, `erl_rescue/5`, the `dynamic_normalize` `foldl`, `rescue_guards` comprehension) is linear in the number of source clauses/aliases and cannot be amplified. The candidate crash sinks are enumerated and ruled out below.

## Risks / caveats

- **Reachability rests on the expander being the sole upstream.** I verified that the only documented public-API paths to this module — `Code.compile_string/eval_string` (parser path) and `Code.eval_quoted` (hand-built-AST path) — both funnel through `elixir_expand:expand` → `elixir_erl_pass:translate` (confirmed in `elixir.erl:448-451`, `quoted_to_erl/4`), and that the expander (`elixir_clauses.erl:426-450`, `expand_catch`/`expand_rescue`) rejects every malformed clause *before* `elixir_erl_try` runs. Both were driven live (see Validation lines). If a future refactor let unexpanded/hostile `try` AST reach `elixir_erl_pass:translate({'try',…})` directly, the incomplete `case Args of` (line 33) and the non-exhaustive `each_clause/2` heads would become reachable `case_clause`/`function_clause` compile-time crashes — but that is a self-inflicted crash of the compiling process (the threat model's "a parse crash kills only the caller"), not a memory-safety or code-execution issue.
- **The `debug_info` legacy path** (comment at lines 30-32) can only carry *already-expanded* clauses that passed validation at their original compile time, so it cannot deliver a 3-arg catch clause; the `[X] | [X,Y]` case remains exhaustive for it.
- Calling the exported `clauses/3` directly with fabricated arguments can crash it, but that is misuse of an internal compiler interface (`elixir_erl_*` modules are undocumented compiler internals) and is out of scope.

### Ruled out

**Incomplete `case Args of [X] -> …; [X, Y] -> … end` (each_clause/2, lines 33-36) — potential `case_clause` DoS.**\
Reason: A catch clause reaching here always has exactly 1 or 2 args because `elixir_clauses:expand_catch/4` errors on any other arity before translation; both surviving arities are matched.\
Axes examined: arg-count 0 / 1 / 2 / 3+ via parser (`compile_string`), arg-count 3 & rescue-arity 2 via hand-built AST (`eval_quoted`), guard-present vs absent (`extract_splat_guards`) | Not examined: none (debug_info sub-path reasoned: it stores only post-expansion, already-validated clauses, so cannot carry a rejected arity).\
Basis: verified.

**Non-exhaustive `each_clause/2` heads (lines 26, 43, 49) — potential `function_clause` DoS.**\
Reason: After expansion a rescue clause is normalised to exactly a variable `{Name,_,Atom}` or an `{in,_,[Var,ListOfAtoms]}` form (`elixir_clauses.erl:453-489`), and catch clauses are tagged `'catch'`; these are precisely the three heads present.\
Axes examined: clause shape var / alias / `in`-list / `in`-underscore / arbitrary-expr / non-list `in` Right / empty `in` list — all via parser and/or `eval_quoted` | Not examined: none.\
Basis: verified.

**`erl_rescue/5` `is_atom(H)` guard + non-atom `Right` (lines 133, 115) — potential `function_clause` on a non-atom alias.**\
Reason: `Right` is `normalize_rescue`d in the expander to `false` (→ `invalid_rescue_clause` error), `[Atom]`, or a list where `lists:all(is_atom, …)` holds (`elixir_clauses.erl:491-494`), so `erl_rescue` only ever iterates atoms.\
Axes examined: `in` Right = single atom, list of atoms, integer `123`, empty list `[]` — driven via parser and `eval_quoted` | Not examined: none.\
Basis: verified (ran `e in 123` → `CompileError`, never reaches this module).

**`erl_rescue_stacktrace_for(_, _, 'Elixir.ErlangError') -> error(badarg)` (lines 91-93) — deliberate crash / dead defensive code.**\
Reason: `dynamic_normalize/3` is only ever called with `?REQUIRES_STACKTRACE` (no `ErlangError`) or `Some` ⊆ `?REQUIRES_STACKTRACE`; `ErlangError` is filtered to the `lists:member` `true` branch in `normalize_rescue/5` (lines 60-62) before this function is reached, so the crash clause never fires and its unreachability carries no security gap (it is an internal assertion, not a bypassed control).\
Axes examined: `ErlangError` present alone, mixed with stacktrace-requiring aliases, and with non-requiring aliases — traced through `normalize_rescue`'s partition | Not examined: none.\
Basis: verified.

**`dynamic_normalize/3` requires a non-empty list (line 75) — potential `function_clause`.**\
Reason: Its two call sites pass the non-empty constant `?REQUIRES_STACKTRACE` or `Some`, and the empty case is handled earlier by the `{[], _} -> []` branch (line 66) so `Some` is never empty at the `{Some, _}` call.\
Axes examined: empty vs non-empty stacktrace-alias partition | Not examined: none.\
Basis: verified.

**`erl_rescue_guard_for/3` generated-guard correctness + catch-all `false` (lines 143-226) — could a wrong guard catch the wrong exception?**\
Reason: Each clause emits a structural guard (`tuple_size`/`element(1,…)`/`==`) matching the corresponding Erlang error term, faithfully reproducing what the user wrote (`rescue e in SomeError`); the catch-all `false` correctly means "no Erlang-level guard, use the Elixir struct guard." The mapping is a language-semantics contract, not an attacker-influenced trust decision, and I confirmed a normal multi-alias rescue behaves correctly at runtime.\
Axes examined: multi-alias rescue, Erlang-error rescue, struct rescue (ran `t1.exs`) | Not examined: exhaustive per-exception-module guard equivalence (correctness, not security).\
Basis: verified (for the security-relevant claim that no guard is attacker-tunable).

**Generated-code construction / "injection" via `Alias` splicing (`rescue_guards` lines 118-121, `build_clause` line 125) — code-execution class.**\
Reason: `Alias` values are spliced only into `==`/`is_map_key` comparison operands of a programmer-shaped guard whose operator atoms are all literals from `erl/2`; an attacker choosing the alias only chooses which exception module the clause matches, exactly as the source `rescue e in Module` intends — no operator or callable is derived from input.\
Axes examined: alias identity/value into guard operands, operator-atom provenance (all literals) | Not examined: none.\
Basis: verified.

**Metadata handling `?ann(Meta)` / `?generated(Meta)` (lines 38, 76, 83-87, etc.) — untrusted line/column metadata.**\
Reason: These macros extract/annotate line info via `elixir_erl:get_ann`/`elixir_utils:generated`; a hostile line number in caller-supplied AST at worst produces a self-inflicted compile-time error in the caller's own `eval_quoted`, crossing no trust boundary and affecting no other tenant.\
Axes examined: line as integer / missing / generated flag | Not examined: extreme/negative line integers via `eval_quoted` (bounded, caller-self-inflicted only).\
Basis: asserted.\
Depends on: `elixir_erl.erl` — `get_ann/1` tolerates or cleanly errors on out-of-range line metadata without a cross-caller effect.

**Resource consumption — `reduce_clauses/5`, `erl_rescue/5`, `dynamic_normalize` `foldl`, `rescue_guards` comprehension (lines 20, 133, 79, 118).**\
Reason: Every loop is a single linear pass over the source's clause/alias list with no nested re-walk or quadratic accumulator; total work is bounded by source size, which is the inherent cost of compiling any source, not an amplification in this file.\
Axes examined: clause-count and alias-count scaling (linear), no `<<Acc/binary,…>>`/list-reverse-in-loop patterns present | Not examined: none.\
Basis: verified.

**Shared mutable state / `#elixir_erl` threading (lines 15-18, 24, 44, 50, 125).**\
Reason: State is a functionally threaded record (`stacktrace`, `counter`, `var_names`); the module makes no process-dictionary/ETS/application-env write, so there is no external-derived global one caller could poison for another.\
Axes examined: all state mutations in the file (record updates only) | Not examined: none.\
Basis: verified.

### Did I verify this?

- **Incomplete `case Args of`**, **non-exhaustive `each_clause` heads**, **`erl_rescue` non-atom guard** — Basis `verified` and honest: I read the upstream gate (`elixir_clauses.erl:426-494`) and the sole-path funnel (`elixir.erl:448-451`), and I *ran* the two in-scope untrusted-input surfaces (parser via `Code.compile_string`, hand-built AST via `Code.eval_quoted`), observing `CompileError`/`SyntaxError` before the module is reached (`ran /opt/elixir/bin/elixir t2.exs` / `t3.exs`). `Not examined: none`; the debug_info sub-path is not an untrusted-input surface (producing debug_info requires a prior successful compile that already validated). No `Depends on` needed.
- **`erl_rescue_stacktrace_for` dead crash**, **`dynamic_normalize` non-empty**, **generated-guard correctness**, **Alias-splice injection**, **resource consumption**, **shared state** — `verified` by reading every call site within this file and, where behavioural, running `t1.exs`; each `Not examined` list is `none` for the security-relevant claim. No cross-file contract is load-bearing.
- **Metadata handling** — kept `asserted`. I did not exercise extreme/negative line integers through `get_ann/1`, and the implementation lives in `elixir_erl.erl`, not this file. The `Depends on:` line names that module and the assumed contract (out-of-range line metadata does not produce a cross-caller effect); the cross-file pass should confirm it. Even if wrong, the worst case is a self-inflicted compile-time crash of the caller that supplied the AST — no boundary crossing — so it would not rise above informational.

