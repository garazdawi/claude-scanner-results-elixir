# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_pass.erl
<!-- sha256: 1c57ce37f0a32c869fccdbd9e04331d316936adb831e458482141117ce961218 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_pass.erl
<!-- sha256: 1c57ce37f0a32c869fccdbd9e04331d316936adb831e458482141117ce961218 -->

_Re-reviewed on 2026-08-28_

# Security review: `elixir_erl_pass.erl`

## Summary
This file is the compiler back-half pass that translates an **already-expanded** Elixir AST into Erlang Abstract Format. Every security-relevant sink is either an allow-list-gated compile-time optimization or by-design generated-code semantics, and reaching any of them requires passing first through `elixir_expand:expand` (arbitrary compile-time code execution). There is no attacker uplift at that boundary.

This pass resolves the two hedged entries the prior report left `asserted` with an "outside this file" dependency. Reading the callgraph neighbours (`elixir_utils.erl:136-155` for `characters_to_list/1`; `elixir.erl` / `elixir_compiler.erl` for the entry ordering) confirmed the *security outcomes*, but neither entry's remaining axis is a two-implementation divergence axis that can be executed here — each is a pure **reachability** question in a non-neighbour module (`elixir_tokenizer`/`elixir_interpolation`; `elixir_expand`). Because reading a neighbour never clears an axis and these axes cannot be *tried*, the two entries cannot honestly reach `Not examined: none`. Following "prefer downgrading," I have moved both out of the hedged-dismissal state into **low-confidence findings** under their sink class. Their practical severity is near-nil (a clean compile-time exception / a no-uplift crash), and that is stated in each. No prior finding is dropped (the first pass raised none).

## Security

### Non-UTF-8 binary into a utf8-typed bitstring element — `build_bitstr/7`, elixir_erl_pass.erl:526 (Class: Resource consumption / deserialisation — malformed UTF-8)
**Upgraded from hedged assumption in first pass.**\
`H` reaches line 526 when `is_binary(H)` and `requires_utf_conversion(Types) == true` (utf8/16/32 element types, lines 534-536), and is passed to `elixir_utils:characters_to_list/1`. I read that callee (elixir_utils.erl:136-143): on a non-list input it delegates to `unicode:characters_to_list/1` and, on `{error,_,_}`/`{incomplete,_,_}`, raises `'Elixir.UnicodeConversionError'` via `conversion_error/3` (elixir_utils.erl:154-155). So the **outcome is safe**: a clean compile-time exception, never memory unsafety, replacement-char corruption, or silent truncation, and output length is source-bounded.

The residual, unverifiable axis is whether a sigil / `Interpol=false` tokenizer path (or a compile-time-assembled `<<255>>::utf8` literal) can actually deliver a non-UTF-8 binary into a utf8-typed element. That reachability lives in `elixir_tokenizer.erl` / `elixir_interpolation.erl` — not callgraph neighbours of this module — and is not a divergence axis I can exercise by pushing a value through two implementations, so I could not try it. I record this as a finding rather than a hedged dismissal solely because the axis is untried; the security consequence, if reachable, is a benign `UnicodeConversionError` compile error (DoS-moot given the code-execution primitive already required to reach translation).\
**Confidence: low.** (Outcome verified safe in-neighbour; only the immaterial reachability axis is untried.)

### Shape-assumption pattern matches — `cond` (line 130), `receive`-after (lines 181-182), `with` (line 193) (Class: crash-on-malformed-AST)
**Upgraded from hedged assumption in first pass.**\
These clauses assume post-expansion invariants: non-empty `cond` clauses (`lists:reverse(Clauses)` then `[H | T]` at line 130), a single-pattern `after` (`{_, _, [FExpr], _, FAfter}` at line 182), and `do` present in `with` (`{Exprs, [{do, Do} | Opts]}` at line 193). I re-read the two *called-by* neighbours and confirmed translation is always preceded by expansion — `elixir.erl` (`quoted_to_erl`: `elixir_expand:expand` then `elixir_erl_pass:translate`) and `elixir_compiler.erl` (`interpret`/`compile`). A malformed hand-built AST reaching `translate/3` directly would be an internal-interface contract violation (out of scope) and confers no uplift over the compile-time code execution required to arrive here.

The untried axis is the exact set of AST shapes `elixir_expand` guarantees. `elixir_expand.erl` is not a callgraph neighbour, and this is a reachability/contract question, not a two-implementation divergence I can execute, so it stays untried. Recorded as a low-confidence finding rather than a hedged dismissal for that reason; the impact if the invariant were ever violated is a `badmatch`/`function_clause` in the compiling process — no uplift over the code execution already needed.\
**Confidence: low.** (Reachability + no-uplift verified in-neighbour; only the immaterial exact-shape axis is untried.)

### Ruled out

**Compile-time `apply(Left, Right, Args)` — `rewrite_strategy/4`, elixir_erl_pass.erl:726** (Class: Code execution)\
Reason: `apply` is gated by `inline_pure_function(Left, Right)` (fixed allow-list of pure `Duration`/`MapSet`/`String`/`Kernel`/`URI`/`Version` functions, lines 775-793) *and* `basic_type_arg(Args)` (only numbers/atoms/binaries/lists/2-tuples of same, lines 796-799), and is wrapped in `try…catch _:_ -> none` (lines 725-730). Result size is bounded by source-literal size, and reaching it already implies compile-time code execution, so zero uplift.\
Axes examined: reachable modules (allow-list only), reachable functions (allow-list only), arg types incl. nested lists/2-tuples, non-2-tuple ASTs (3-tuples fall to `basic_type_arg(_) -> false`), try/catch coverage, purity of allow-listed funcs | Not examined: pathological CPU inside an allow-listed pure fun (e.g. `String.split` on a maximal source-literal binary) — bounded by source size and by the compiler already executing attacker macros.\
Basis: verified. Confidence: high.

**Compile-time `Left:'__duration__!'(Opts)` — `rewrite_strategy/4`, elixir_erl_pass.erl:714** (Class: Code execution)\
Reason: `Left` is constrained by the clause guard to exactly `Date`/`DateTime`/`NaiveDateTime`/`Time` (lines 705-710), the function name is the fixed `'__duration__!'`, `Opts` passes `basic_type_arg/1`, and it is `try…catch`-wrapped; same no-uplift reasoning as above.\
Axes examined: reachable modules (4-way guard), function name (literal), arg shape (`basic_type_arg`), try/catch | Not examined: none.\
Basis: verified. Confidence: high.

**Runtime `apply(Atom, Fun, [])` — `no_parens_remote/2`, elixir_erl_pass.erl:827** (Class: Code execution)\
Reason: generated-code runtime support for the *deprecated* no-paren `map.field` notation (emitted at line 259); `Fun` is the source-fixed field atom and `Atom` is the runtime receiver — invoking `Receiver.field/0` is precisely the historical language semantics of `x.field`, emitted only for that deprecated syntax with a deprecation warning. Direct calls to the exported `no_parens_remote/2` are internal-interface use (out of scope).\
Axes examined: `Fun` origin (source-fixed literal), receiver branches for nil/false/true (`{error,{badmap,_}}`)/map-without-key (`{error,{badkey,_}}`)/other (`{error,{badmap,_}}`)/atom (apply) | Not examined: none.\
Basis: verified. Confidence: high.

**Deprecation-warning format strings — `no_parens_remote/2` line 820, `parens_map_field/2` line 835** (Class: Template/interpolation — format string)\
Reason: format strings are compile-time constants; interpolated values reach only `~ts` data placeholders (`elixir_aliases:inspect(Atom)`, `Fun`/`Key` atoms), so no format-directive injection is possible; `'Elixir.IO':warn_once(?MODULE, …, 3)` keyed on `?MODULE` bounds volume.\
Axes examined: format-string constness, `~ts` argument types (atom/binary/iolist), directive injection via interpolated values, warning-flood volume | Not examined: none.\
Basis: verified. Confidence: high.

**Atom creation via `elixir_utils:macro_name/1` — `super` translation, elixir_erl_pass.erl:90** (Class: Resource consumption — atom exhaustion)\
Reason: I read the callee: `macro_name(Macro) -> list_to_atom("MACRO-" ++ atom_to_list(Macro))` (elixir_utils.erl:27-28). Its argument `Name` comes from `lists:keyfind(super, 1, Meta)` (line 86) — an existing function-name atom of the current module. A module's function-name set is finite and already atomised, so no unbounded atom growth.\
Axes examined: `Name` origin (module function name, pre-existing atom), callee body (verified in elixir_utils.erl:27-28), atom-namespace growth bound | Not examined: none.\
Basis: verified. Confidence: high.

**`String/List.to_existing_atom` inlining — `translate_remote/5`, elixir_erl_pass.erl:637-661** (Class: Resource consumption — atom exhaustion)\
Reason: conversions are the safe direction — `atom_to_binary`/`atom_to_list` on atoms already present in the source-literal `List` (guarded by `lists:all(fun is_atom/1, List)`, line 641); the generated `case` maps input strings to *existing* atoms and otherwise defers to `__to_existing_atom__`, creating no new atoms.\
Axes examined: conversion direction (atom→string), atom-list origin (source literal), fallback path | Not examined: none.\
Basis: verified. Confidence: high.

**State-discarding in `lists:member` inlining — `translate_remote/5`, elixir_erl_pass.erl:603,608** (Class: Shared mutable state / concurrency — var-hygiene)\
Reason: elements translated with the un-threaded `S` are restricted by `optimize_list_membership/2` to ≤32 numbers/atoms/binaries (lines 695-701), which allocate no variables and have no side effects, so discarding the returned state cannot cause variable capture; the receiver `Expr` state *is* threaded (`S1`→`S3`). Correctness-only at most.\
Axes examined: element types (literals only), var allocation by element translation, threading of the receiver expression | Not examined: none.\
Basis: verified. Confidence: high.

**`maps:merge`/`map_field` construction — `translate_remote/5`, elixir_erl_pass.erl:621-636** (Class: Round-trip integrity)\
Reason: `Pairs1 ++ Pairs2` places the second map's fields last, matching `maps:merge/2` "right wins" semantics under Erlang map-literal last-key-wins; no meaning change across the optimization.\
Axes examined: key-collision precedence (last-wins vs merge semantics), map-vs-update variants | Not examined: none.\
Basis: verified. Confidence: high.

**Recursive translation depth — `translate/3`, `translate_list/4`, `translate_block/4`, `translate_with_do/5`, `build_cond_clauses/4`** (Class: Resource consumption — stack)\
Reason: recursion depth mirrors AST nesting, which the tokenizer/parser already had to build (and recurse over) to reach expansion; no amplification occurs here, and reaching it requires compile-time code execution.\
Axes examined: recursion-depth source (AST nesting = source nesting), amplification vs parser | Not examined: none.\
Basis: verified. Confidence: high.

Sinks searched for and **absent** in this file: `binary_to_term`, `list_to_atom`/`binary_to_atom` on external strings, `open_port`/`os:cmd`, any `file:`/`filelib:` op, `xmerl_scan`, ETS match-spec construction, `crypto:*`, NIF loads, network calls, registered-name races.

### Did I verify this?

- Compile-time `apply` / `__duration__!` / runtime `apply` / warnings / `to_existing_atom` / `lists:member` / `maps:merge` / recursion / `macro_name`: **verified** — read the gating code (`inline_pure_function/2`, `basic_type_arg/1`, `optimize_list_membership/2`, clause guards) in this file and the callee `elixir_utils:macro_name/1` (elixir_utils.erl:27-28). `Not examined` reads `none` on each.
- `characters_to_list` on bitstring literals: **outcome verified in-neighbour, entry now a low-confidence finding.** I read `elixir_utils:characters_to_list/1` (elixir_utils.erl:136-155) and confirmed invalid/incomplete UTF-8 raises a clean `UnicodeConversionError`. The one remaining axis — tokenizer reachability of a non-UTF-8 binary into a utf8-typed element — lives in `elixir_tokenizer`/`elixir_interpolation` (non-neighbours) and is a reachability, not divergence, axis, so it cannot be *tried* here and reading cannot clear it. Per the re-review rule I moved this from a hedged `asserted` dismissal to a low-confidence finding rather than assert `Not examined: none` I did not earn.
- Shape-assumption crashes: **reachability + no-uplift verified in-neighbour, entry now a low-confidence finding.** I re-read the two called-by neighbours (`elixir.erl`, `elixir_compiler.erl`) confirming `elixir_expand:expand` precedes `translate/3`. The exact-shape-guarantee axis (`elixir_expand.erl`, non-neighbour) is a contract/reachability axis, not executable as a two-implementation comparison, so it stays untried; the entry is a low-confidence finding for that reason, not a hedged dismissal.

### Dropped in re-review

Nothing dropped. The first pass raised no findings, and no ruled-out entry was removed. The two entries the first pass left `asserted` with the "outside this file" hedge (`characters_to_list`, shape-assumption crashes) have been **converted to explicit low-confidence findings** this pass — the safe direction under the re-review rule — because their sole remaining axis is an untried reachability axis in a non-neighbour module that cannot be executed here. Their verified-in-neighbour security outcomes (clean `UnicodeConversionError`; no-uplift crash) are stated in each finding, so the practical severity remains near-nil; the hedge phrasing has been removed.

## Risks / caveats
- This audit treats the compiler pipeline structure as the trust boundary: reaching `elixir_erl_pass` requires `elixir_expand:expand` (macro execution), re-confirmed for both documented entry points this pass. A future refactor exposing `translate/3` on a non-expanding path (e.g. translating raw untrusted debug-info AST without re-validation) would materially raise the two low-confidence findings above and would require re-evaluating the compile-time `apply` dismissals.
- The two low-confidence findings each rest on an untried reachability axis in a non-neighbour module (`elixir_tokenizer`/`elixir_interpolation`; `elixir_expand`). Neither axis is a two-implementation divergence, so neither could be exercised in this sandbox; a later pass with those modules in scope should either exhibit a reaching input or retire the finding.
- The runtime `apply(Atom, Fun, [])` in `no_parens_remote/2` is deprecated language behaviour, not a compiler bug; application code that routes attacker-controlled runtime values into the receiver of a no-paren `x.field` expression can invoke `Receiver.field/0` — an application-level concern, not this file's responsibility.
