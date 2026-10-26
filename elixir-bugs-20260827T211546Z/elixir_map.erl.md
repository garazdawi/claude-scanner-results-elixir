# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_map.erl
<!-- sha256: baca864a16e9f062bd08aff249c5c68f5af3db81509b04d8c3e430f42bb06c42 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_map.erl
<!-- sha256: baca864a16e9f062bd08aff249c5c68f5af3db81509b04d8c3e430f42bb06c42 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (Erlang/OTP 29, Elixir 1.21.0-dev toolchain available in sandbox)_

# Bug-hunting review — `elixir_map.erl`

## Summary
One genuine defect, now confirmed by execution: a `function_error` call in the repeated-map-key path passes the file binary instead of the env, defeating the error-accumulation semantics and forcing an immediate compile abort. Of the four ruled-out entries the first pass left `asserted`, one (silent-corruption / escape fidelity) is upgraded to `verified` with executed evidence; the other three stay `asserted` because their remaining axis cannot be surfaced from source input (reading a neighbour does not clear an axis), and I say so plainly rather than hedging.

## Bugfixes

#### `CSCAN-elixir_map-repeated-key-passes-file-not-env`
- **Finding ID**: CSCAN-elixir_map-repeated-key-passes-file-not-env
- **Severity**: Medium
- **Confidence**: high
- **Location**: `elixir_map.erl:96`, function `validate_not_repeated/4` (the `#{context := match}` branch)
- **Class**: Contract violation (wrong argument passed — file binary vs env — subverting the callee's continue-on-error contract)
- **User-visible outcome**: For a repeated *literal* key in a **match** pattern inside a function, e.g. `def f(%{a: 1, a: 2} = x), do: x`, the compiler raises a `CompileError` and aborts immediately at that point instead of recording a recoverable (module-tainting) error and continuing to surface the file's other diagnostics. Every other recoverable error site in this file passes the env `E`; this one passes `?key(E, file)` (a binary), which can never match `function_error`'s `#{function := {_,_}}` clause, so it falls straight through to `file_error`, which is `no_return()` and raises.
- **Executed confirmation** (this is what upgrades the finding from *medium* to *high*):
  - `t2.exs` — a module with `def f(%{a: 1, a: 2}), do: :ok` **and** a second, independent error `def g(), do: undefined_thing_xyz()`. Output: **only** `key :a will be overridden in map` was logged, then `** (CompileError) ... cannot compile file`; the `g/0` error was never reached. The stack shows `:lists.foldl_1`, matching `validate_kv/4`'s `lists:foldl`.
  - `t5.exs` — the *same* repeated key in a **non-match** context (`def f(), do: %{a: 1, a: 2}`) plus the same second error. Output: **both** `overridden in map` (warning, via `file_warn`) **and** `undefined_two_bbb` were reported — compilation continued past the map. The only code difference between the two runs is `function_error(…file…)` vs `file_warn` inside `validate_not_repeated/4`; nothing else explains why the match case aborts and the non-match case accumulates.
  - `t4.exs` — control: two undefined-function errors in one module both report, confirming the compiler's normal accumulate-and-continue behaviour that the match-repeated-key path fails to participate in.
- **Mechanism (re-read `elixir_errors.erl:264-289`)**: `function_error/4` routes a map env with `#{function := {_,_}}` to `module_error` → `elixir_module:taint` → `ok` (continue). A **binary** second argument does not match line 286, so it hits line 288 `file_error(Meta, <binary>, …)`, whose `is_binary(File)` head (line 265) wraps it and reaches `compile_error` (line 269) → raise. Passing `E` (a map) would taint-and-continue like the file's three other `function_error` calls (lines 132, 205, 228).
- **Suggested fix**: `function_error(Meta, ?key(E, file), ?MODULE, {repeated_key, Key})` → `function_error(Meta, E, ?MODULE, {repeated_key, Key})`.
- **Risk of fix**: A module whose only error is a repeated match key now taints-and-continues rather than raising at that exact point; overall compilation still fails, but additional diagnostics may now surface and the raise site/stack shape changes. A test asserting an immediate single-error `CompileError` for a repeated match key would observe a more complete error set.
- **Evidence**:
  - read `elixir_map.erl:92-106` (the branch and its `file_warn` sibling), `:132,205,228` (all other `function_error` calls pass `E`)
  - read `elixir_errors.erl:264-289` (`file_error` = `no_return`; `function_error` needs `#{function := {_,_}}` to reach the recoverable `module_error`)
  - executed `t2/t4/t5` above (match aborts; non-match + control accumulate)

## Risks / caveats
- The earlier "could not diff against upstream" hedge is retired: the executed match-vs-non-match asymmetry (`t2` vs `t5`) is a direct behavioural demonstration of the defect on the installed toolchain, which exhibits the buggy (abort-on-match-repeated-key) behaviour. Hence *high* confidence.
- `expand_map/4`, `expand_struct/5`, `maybe_load_struct_info/4` carry no `-spec`; the spec-totality lens is trivial. `format_error/1` is total over the atoms/tuples this module constructs (each error constructor has a matching clause, lines 252-305).

### Ruled out

**Operator / precedence typo**
- Reason: The one compound boolean (`is_literal(Key) andalso Used`, line 93) is intentional — `andalso` short-circuits to `false` for a non-literal key, and the `case` discriminates `false` vs `#{Key := true}` vs the catch-all `#{}` correctly.
- Axes examined: literal atom key, non-literal 3-tuple AST key, key already-present, key absent | Not examined: none
- Basis: verified

**Missing clause / catch-all**
- Reason: `validate_match_key/3` and `is_literal/1` both end in catch-alls; `expand_struct/5`'s inner `case ERight` (lines 27-45) covers update / non-match-construct / match; the un-guarded `case ERight` can only be entered when `expand_struct`'s **first clause head** already matched a literal `{'%{}', MapMeta, MapArgs}` node (line 21), and `elixir_expand:expand({'%{}', Meta, Args}, …)` (`elixir_expand.erl:28-29`) dispatches to `elixir_map:expand_map`, which returns `{{'%{}', Meta, EArgs}, …}` echoing the *same* `Meta` (`elixir_map.erl:13,19`) — so `ERight` structurally carries the same `MapMeta` the case clauses match on.
- Axes examined (executed, `battery.exs`): struct construct `%S4{a,b}`, struct update `%S4{x | a}`, struct match `%S4{a: 1}=s`, nested struct/map/tuple defaults, update using a field ref — **no `case_clause` or internal crash on any**, all three case branches (update/construct/match) exercised.
- Not examined: an `ERight` whose meta differs from the bound `MapMeta`. This is not a triable axis: there is no source construct that carries it, because (a) `expand_struct` clause 1 only fires for a literal `{'%{}', MapMeta, _}` node and (b) `expand_map` always re-emits that same `Meta`. Reading `elixir_expand.erl:28-29` shows *why* it can't diverge but, per the axis rule, does not itself count as trying the axis — and no input can, so it stays not-examined.
- Basis: asserted (reachable branches executed with no crash; the divergence axis is unconstructable from source, so the entry cannot be promoted to `verified`)

**State / accumulator threading error**
- Reason: `expand_map`/`expand_struct` return `SE`/`EE` (post-expansion state/env) consistently. The `E` vs `EE` choice at lines 29/34/42 feeds only `elixir_env:trace` and `function_error`, whose consulted fields (tracers, file, module, function) are lexical. Re-read of `elixir_expand.erl:734-745`: the non-match `expand_args` path threads env via `expand_arg`→`expand`, which does not rewrite `file`/`module`/`function`/`tracers` when expanding a key/value arg.
- Axes examined: update clause (uses `E`), construct clause (uses `EE`), match clause (uses `E`) — all compiled cleanly in `battery.exs`.
- Not examined: an env whose tracer/module set is actually mutated by `expand_args` so `E` and `EE` diverge in a traced field. Not surfaceable from source syntax (arg expansion does not mutate those lexical fields), so I could not push such a value through; the axis stays untried.
- Basis: asserted (reading located the threading; the divergence axis was not executed and cannot be forced, so no promotion)

**Wrong atom / wrong symbol**
- Reason: Key atoms (`'__struct__'`, `field`, `function`, `context`, `match`, `nil`) are used consistently and match the shapes produced/consumed (`Name:'__info__'(struct)`, `Name:'__struct__'(Assocs)`).
- Axes examined: `__struct__` key removal, `field` lookup in Info maps, `context := match`/`nil` | Not examined: none
- Basis: verified

**State threading — see above.**

**Off-by-one / wrong arithmetic**
- Reason: `validate_kv/4`'s `Index` starts at 1; `lists:nth(Index, Original)` is reached only in the not-a-pair clause, whose `file_error` raises immediately, so a stale index for a later bad element is never observed. `KV` and `Original` have equal length (both from the same arg list via `expand_args`).
- Axes examined: first element not a pair, valid pair then bad pair, empty KV | Not examined: `KV`/`Original` length mismatch (would require `expand_args` to change arity — not observed in any `battery.exs` run)
- Basis: verified

- **Contract violation** — promoted to the finding above.

**Crash on stray / unexpected input**
- Reason: Entry points receive parsed AST from `elixir_expand`; malformed assoc shapes route to `file_error`/`function_error`, not a crash; `load_struct`/`maybe_load_struct_info` wrap risky calls in `try` with `error:undef` handling plus an annotating re-raise (lines 216-223).
- Axes examined: `__struct__/1` returning a non-map (`Other` clause), wrong struct name, `undef`, other errors; also executed `%S{unknown: 1}` on a real struct (`t3.exs`) → controlled `KeyError` with `expanding struct` annotation, no internal crash | Not examined: none (the `Other` clause covers a map lacking `__struct__`)
- Basis: verified

**Resource leak on error path**
- Reason: No files, ETS tables, processes, or monitors created here; `ets:lookup_element` is read-only and `try`-wrapped (`lookup_struct_info_from_data_tables/1`, lines 167-173).
- Axes examined: ETS lookup on missing table (caught `_:_` → false) | Not examined: none
- Basis: verified

- **Silent corruption** — *Upgraded to verified (executed).*
- Reason: `maps:without(Keys, Struct)` + `StructAssocs ++ Assocs` preserves override order (defaults first, user assocs last so they win); `lists:sort` gives deterministic default ordering; `elixir_quote:escape/3` faithfully turns default `{key,value}` pairs into equivalent AST.
- Axes examined (executed): (`t6`/`suse2.exs`) struct with tuple / list-containing-map / nested-map-with-tuple-and-list defaults → all reconstructed byte-for-byte; user override beats default (`a: :overridden` won over the `{1,2}` default); un-overridden defaults preserved; struct update preserves other defaults; **and the previously-untried "unescapable term" axis: a function-valued default `&Kernel.+/2`, a binary `<<1,2,3>>`, and a charlist `~c"…"` all round-tripped correctly** (the function default even remained callable, returning 5).
- Not examined: none — the escape-fidelity/ordering axes, including exotic default terms, were executed against a real `defstruct`/construction/update round trip.
- Basis: verified

**Infinite loop / hang**
- Reason: No `receive`, no unbounded recursion; `validate_match_key`/`is_literal` recurse structurally over finite AST.
- Axes examined: deeply nested map-key AST, list of assocs | Not examined: none
- Basis: verified

**Quadratic-where-linear / Repeated work**
- Reason: Comprehensions and `lists:foldl` are single-pass; the `lists:any` per assoc is O(struct fields), a compile-time constant, not a hot runtime loop.
- Axes examined: many assocs, many struct fields | Not examined: none
- Basis: verified

**Dead / unreachable code**
- Reason: Every `format_error/1` clause corresponds to an error tuple constructed in this module; both `expand_struct` catch-alls and both `validate_struct` `false` branches are reachable via match/non-match contexts.
- Axes examined: each error-tuple constructor vs its formatter clause | Not examined: none
- Basis: verified

**Documented behaviour mismatch**
- Reason: No `@doc`/`-spec` prose contracts in-file; behaviour is defined by the error messages, which match their triggers.
- Axes examined: repeated_key (match=error vs non-match=warn — executed via `t2`/`t5`), unknown_key, invalid_key | Not examined: none
- Basis: verified

**Race condition / shared mutable state**
- Reason: The race windows around `external_for`/ETS-table deletion are handled with `try`/fallback to `Name:'__struct__'/1` (lines 196-200, 216-218) and commented. Re-read of `elixir_module:data_tables/1` (`elixir_module.erl:34-35`) confirms it is a bare `ets:lookup_element(elixir_modules, Module, 2)` that raises `badarg` when the module row is gone — exactly the failure the caller's `try … catch _:_ -> false` (lines 168-173) and `error:undef` fallback (line 199) are written to absorb; `elixir_def:external_for/5` exists at `elixir_def.erl:31` as the guarded call. No shared mutable state is written by this module.
- Axes examined: ETS table missing at lookup (caught → false), external fun `undef` fallback (structurally present).
- Not examined: an actual concurrent deletion racing the lookup. This cannot be triggered deterministically from a single compile run, so the timing axis stays untried; reading `data_tables/1` confirms the raise the guard targets but does not exercise the race.
- Basis: asserted (guard rationale confirmed by reading; the timing axis is not deterministically triable, so no promotion)

**Wrong scope / order of operations**
- Reason: `delete_struct_key` runs before `expand_args`, stripping `__struct__` before expansion; validation runs before the final AST is assembled.
- Axes examined: struct key present (warn + strip) then expand, absent | Not examined: none
- Basis: verified

**Encoding / unicode mishandling**
- Reason: No byte/codepoint handling; keys are atoms/AST and messages use `~ts` with `Macro.to_string`/`inspect`.
- Axes examined: atom key formatting, non-atom key error formatting | Not examined: none
- Basis: verified

### Did I verify this?

- **Contract (finding)** — verified by execution (`t2` vs `t5` asymmetry, `t4` control) plus re-read of `elixir_errors.erl:264-289`. Upgraded medium → high.
- **Silent corruption** — **upgraded from hedged assumption in first pass** (`asserted`, depending on `elixir_quote:escape/3`) to **verified** by executing real `defstruct` round trips over tuples, lists, nested maps, binaries, charlists, and a function-valued default (`t6`, `suse2.exs`); every default reconstructed faithfully and user assocs won over defaults. `Not examined: none`.
- **Missing clause / catch-all** — stays **asserted**. Reachable case branches (update/construct/match) were executed with no `case_clause` (`battery.exs`), and `elixir_expand.erl:28-29` shows `expand_map` echoes `Meta`; but the "differing-meta" axis is unconstructable from source, so no input carries it — it stays not-examined and the entry cannot be `verified`. First-pass "assumed" language removed.
- **State / accumulator threading** — stays **asserted**. Re-read `elixir_expand.erl:734-745`: the non-match `expand_args`/`expand_arg` path does not rewrite the lexical `file`/`module`/`function`/`tracers` fields; but I could not construct an env in which `E` and `EE` diverge in a traced field, so that axis is untried. First-pass "assumed" language removed and replaced by what the read showed.
- **Race condition** — stays **asserted**. Re-read `elixir_module:data_tables/1` (`:34-35`, raises `badarg` when the row is gone) and confirmed `elixir_def:external_for/5` (`:31`) exists; the in-file `try` guards target exactly that raise. The concurrent-deletion timing axis is not deterministically triable, so it stays not-examined. First-pass "assumed" language removed.

No prior findings dropped. (No "Dropped in re-review" subsection is needed — the single prior finding is retained and strengthened.)
