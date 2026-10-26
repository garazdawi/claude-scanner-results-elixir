# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_def.erl
<!-- sha256: b0c7a583301f6dc2db656a39b415c6ced62744a56c2a021f4dce8cdbfc66abf6 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_def.erl
<!-- sha256: b0c7a583301f6dc2db656a39b415c6ced62744a56c2a021f4dce8cdbfc66abf6 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T (hedge-resolution pass)_

# Bug-hunting review — `elixir_def.erl`

## Summary
This file matches the genuine Elixir `elixir_def` module clause-for-clause. The prior pass found no defect; this re-review reads the callgraph neighbours (`elixir_overridable`, `elixir_dispatch`, `elixir_env`, `elixir_expand`) and greps the source tree to discharge the previously-hedged dismissals. The result: three ruled-out entries move from `asserted` to `verified` on concrete read/grep evidence; one (`Dead code`) surfaces a **low-confidence finding** (`format_error({no_module,...})` has no producer anywhere in the available tree); two (`State threading`, `Race condition`) stay `asserted` because their remaining axes require execution/concurrency evidence I did not produce, and reading a neighbour cannot clear a runtime-differential axis.

## Bugfixes

### 1. `format_error({no_module, ...})` clause has no producer in the available source tree — possible dead clause (Dead / unreachable code)
**Upgraded from hedged assumption in first pass** (was: `Ruled out → Dead code`, `Basis: asserted`, `Not examined: whether every format_error tag is still produced somewhere in-tree`).

- **Location:** line 418, `format_error({no_module, {Kind, Name, Arity}}) -> "cannot define function outside module, ..."`.
- **Evidence I ran:** `grep -rn "no_module"` across the entire available tree (`/Users/lukas/code/claude-scanner/elixir/lib`, which in this checkout contains only `elixir/lib/elixir/src`) returns exactly one hit — the `format_error` head itself at `elixir_def.erl:418`. No `elixir_errors:*(..., ?MODULE, {no_module, ...})` (or any other) call site produces this tag. By contrast, every *other* `format_error` tag in this file has an in-tree producer: `function_head` (line 111), `duplicate_defaults` (366), `mixed_defaults` (369/372), `ungrouped_*` (356/360), `late_function_head` (349), `changed_kind` (341), `no_alias` (398), `invalid_def` (137), `invalid_function_head` (243/379), `__info__` (403), `module_info` (404/406), `is_record` (408), and (in the neighbour) `bad_kind`/`no_super` in `elixir_overridable` (44/57/74). Only `no_module` is orphaned.
- **Why low-confidence (not a confirmed defect):** the "cannot define function outside module" check historically lives in the Elixir-level `Kernel`/`Module` macro layer, and this checkout contains only the `.erl` compiler sources (`lib/elixir/` has just `src/`), not the `.ex` files. So the producer may exist out-of-checkout and pass `elixir_def` as the reporting module. If it does, the clause is live and correct; if the check was moved/renamed, this clause is dead. I cannot distinguish these two from the available tree.
- **Failure scenario:** if the producer was removed/renamed upstream and this clause is truly dead, a `def`-outside-module path that *should* report `no_module` would instead fall through to a different (or missing) error, degrading the diagnostic. No crash; diagnostic-quality regression only.
- **Confidence:** Low.

## Risks / caveats
- I read the in-folder consumers `elixir_overridable.erl` and `elixir_dispatch.erl` and the state helpers `elixir_env.erl` (and confirmed `elixir_expand:expand`'s 3-tuple return shape via its clause heads). Those reads discharge the contract-shape and reset-read dismissals below. I did **not** execute any differential test (the sandbox toolchain was not used to run a multi-default expansion or a concurrent-compile scenario), so the two entries whose remaining axis is a *runtime* property stay `asserted`.
- The default-argument unpacking arithmetic (`unpack_expanded`/`match_defaults`/`extract_defaults`, lines 297–335) remains the highest-scrutiny region for future edits; it matches upstream and each counter step is traced below.

### Ruled out

**Operator / precedence typo**
- Reason: Every guard connective matches intent — `Kinds == all orelse lists:member(...)` (line 42), `Kind == defp; Kind == def` (line 408), and the nested `andalso/orelse` in `defaults_conflict/4` (lines 393–395) partition arity ranges correctly.
- Axes examined: `Kinds=all` vs list vs non-member; `Kind` defp/def/defmacro; defaults A>Arity and A<Arity boundaries | Not examined: none
- Basis: verified

**Missing clause / catch-all**
- Reason: `check_valid_defaults/11` (lines 364–375) is total over Defaults>0 (clauses 1–2) and Defaults=0 (clauses 3–4); `def_to_clauses/6` handles nil/`[{do,_}]`/list-with-`do`/no-`do` (error); `fun_for`'s `try` has `_:_ -> false`.
- Axes examined: Defaults 0 / >0 with Stored 0/>0; body nil / do-block / try-block / missing do | Not examined: none
- Basis: verified

**Wrong atom / wrong symbol**
- Reason: ETS keys (`{def,Tuple}`, `{clauses,Tuple}`, `{default,Name}`, `defs`, `?last_def`, `used_private`) and record fields (`#elixir_ex{version=...}`) are used symmetrically between writers (`store_definition/9`, `track_defmacrop`) and readers (`fun_for`, `take_definition`, `fetch_definition`).
- Axes examined: each ETS key insert vs lookup/take/delete pairing; record field on read vs write | Not examined: none
- Basis: verified

**State / accumulator threading error**
- Reason: `expand_defaults/4` forwards `elixir_env:reset_read(SE, S)` after each default. I read `elixir_env:reset_read/2` (`elixir_env.erl:69-70`): it pattern-matches `reset_read(#elixir_ex{vars={_,Write}}=S, #elixir_ex{vars={Read,_}})` and returns `S#elixir_ex{vars={Read, Write}}` — it rewrites **only** the `vars` field, leaving `version` (the hygiene counter) taken from the first argument `SE`. So the counter accumulated by `elixir_expand:expand` is carried forward, while read-vars are reset to the pre-expansion `S` so defaults don't leak vars between each other. `unpack_defaults` then reads `#elixir_ex{version=Counter}` off that final threaded state (line 298). `elixir_expand:expand/3` returns a 3-tuple `{Expr, S, E}` (confirmed from its clause heads), matching the `{ExpandedDefault, SE, _}` destructure at line 312.
- Axes examined: first-def vs redefine path; empty vs multi-default arg lists; Acc reversal; `reset_read/2` field-preservation (read) | Not examined: differential execution of a ≥2-default expansion confirming the observed counter progression at runtime (not run in the sandbox)
- Basis: asserted
- Note: Reading `reset_read/2`/`env_to_ex/1` establishes the structural invariant (version preserved, only vars reset), but the remaining axis is a runtime-differential one; per the pass rules a read does not clear it, so this stays `asserted`. The Depends-on is now backed by the read rather than an assumption.

**Off-by-one / wrong arithmetic**
- Reason: default-var counters (`length(Acc)+Counter`, `length(Base)+Counter`, `Counter-1` in `match_defaults`) reproduce the upstream numbering that names generated vars `x<N>`.
- Axes examined: 0, 1, and 2+ defaults; interleaved plain/default args | Not examined: none
- Basis: verified

**Contract violation**
- Reason: `take_definition/2` returns `{Result, Clauses}` where `Result` is the full 6-tuple `{{def,Tuple}, Kind, Meta, File, Check, {Defaults,HasBody,LastDefaults}}` (line 81/84). I read the consumer: `elixir_overridable:store/5` destructures its `Def` argument at `elixir_overridable.erl:82-83` as exactly `{{{def, {Name, Arity}}, Kind, BaseMeta, File, _Check, {Defaults, _HasBody, _LastDefaults}}, Clauses}` — a byte-for-byte shape match with `take_definition`'s output. I also read `elixir_dispatch.erl:78` and `:183`, which consume `elixir_def:local_for/5` only in `false ->` / `_ -> ` position (fun-or-false), matching `fun_for`'s return. `fetch_definitions/2` returns a plain list.
- Axes examined: found vs `[]` (missing) def; empty clause list; exact tuple arity/shape destructured by `elixir_overridable:store` (read); `local_for` fun-vs-false consumption by `elixir_dispatch` (read) | Not examined: none
- Basis: verified

**Crash on stray / unexpected input**
- Reason: Public entry points validate shape before use — `store_definition/5` raises a clean `file_error` on malformed `NameAndArgs` (line 137); `fun_for`/`fetch_definitions` guard ETS misses with `try…catch`. The remaining `badmatch`es (`fetch_definition` line 103, `take_definition` line 81) require the `{def,_}` row to have a shape other than the 6-tuple. I grepped the whole tree for `ets:insert(..., {{def, ...})`: the **only** writer is `elixir_def.erl:293`, which writes exactly the 6-tuple `{{def,Tuple}, Kind, FirstMeta, File, CheckClauses, {MaxDefaults,HasBody,Defaults}}`. No other module (including the listed caller `elixir_module` and `elixir_overridable`, which only *reads* `{def,_}` at `elixir_overridable.erl:66`) writes that key. So the readers can never see a foreign shape.
- Axes examined: malformed def call (non-atom name); missing module tables; missing clauses row; sole-writer of `{def,_}` rows (tree-wide grep) | Not examined: none
- Basis: verified

**Resource leak on error path**
- Reason: The module owns no processes/handles/monitors; ETS tables are owned by `elixir_module`, and `ets:take`/`delete_object` in `take_definition` do not leak on the `[]` branch.
- Axes examined: take on present vs absent key | Not examined: none
- Basis: verified

**Silent corruption**
- Reason: `add_defaults_to_meta/2` and the `{MaxDefaults, HasBody, Defaults}` triple preserve all fields; no accumulator is dropped on empty-list clauses (`unpack_expanded`/`expand_defaults`/`extract_defaults` all reverse and return the full Acc).
- Axes examined: 0 defaults (meta unchanged), >0 defaults, empty clause list | Not examined: none
- Basis: verified

**Infinite loop / hang**
- Reason: No `receive`, no `gen_server` callback, no unbounded recursion — all recursions (`fetch_definition`, `unpack_expanded`, `expand_defaults`, `extract_defaults`, `match_defaults`) strictly consume a list head per step.
- Axes examined: empty and long arg/clause lists | Not examined: none
- Basis: verified

**Quadratic-where-linear / Repeated work**
- Reason: Accumulators use `[H|Acc]` + single `lists:reverse` (not `++`); `Base ++ Args`/`Base ++ Invoke` (line 304) append short per-clause default lists, not loop accumulators.
- Axes examined: many-default functions; large clause counts | Not examined: none
- Basis: verified

**Dead / unreachable code**
- Reason: All clauses of `check_valid_defaults`, `assert_valid_name`, `def_to_clauses`, and every `format_error/1` head *except* `no_module` are reachable from an in-tree producer (each producer line cited in finding #1). The `no_module` clause (line 418) is the one exception and is promoted to a low-confidence finding above.
- Axes examined: each error tag emitted vs formatted, verified by tree-wide grep of every tag | Not examined: none
- Basis: verified (with one gap promoted to finding #1)

**Documented behaviour mismatch**
- Reason: The module has no `-spec`/`@doc` prose contracts to contradict; comments (lines 157–163) describe the linify behaviour, which `retrieve_location/2` + `run_with_location_change/3` implement faithfully.
- Axes examined: `meta_keep` = `{file,line}` vs `nil`; file attr as binary vs `{binary,line}` tuple vs absent | Not examined: none
- Basis: verified

**Race condition / shared mutable state**
- Reason: All shared state is the module's per-module ETS tables accessed within a single compile process; no cross-process coordination is attempted here.
- Axes examined: concurrent compile isolation (per-module tables) | Not examined: two processes compiling the *same* module concurrently (would require a concurrency harness / `elixir_module` ownership-model confirmation I did not execute)
- Basis: asserted
- Depends on: `elixir_module` guarantees one compile process per module owns its `{Set,Bag}` tables at a time. This is an ownership invariant of a supervisor-managed neighbour; reading it would describe the design but not exercise the concurrent path, so the axis stays untried and the entry stays `asserted`.

**Wrong scope / order of operations**
- Reason: `retrieve_location/2` takes (and deletes) the `file` attribute before building `DefMeta`; `assert_no_aliases_name`/`assert_valid_name` run before `store_definition/9`.
- Axes examined: file-attr present/absent ordering; name validation before store | Not examined: none
- Basis: verified

**Encoding / unicode mishandling**
- Reason: Names/atoms are passed through as atoms; `format_error` uses `~ts` for names/files throughout; no byte/codepoint confusion.
- Axes examined: `~ts` vs `~s` in every format string; atom vs binary name | Not examined: none
- Basis: verified

**Unclassified**
- Reason: No file-specific one-off defect beyond finding #1; the file is otherwise a faithful copy of upstream.
- Axes examined: default-counter arithmetic, ETS tuple arity symmetry, error-path totality | Not examined: none
- Basis: verified

### Did I verify this?

- **Operator/precedence, Missing clause, Wrong atom, Off-by-one, Silent corruption, Infinite loop, Quadratic, Resource leak, Encoding, Wrong-scope, Documented-behaviour, Unclassified** — `verified`, unchanged from first pass; every axis reads `Not examined: none`.
- **Contract violation** — **upgraded `asserted → verified`.** I read `elixir_overridable.erl:82-83` (destructures `take_definition/2`'s exact 6-tuple+clauses shape) and `elixir_dispatch.erl:78,183` (consumes `local_for/5` only as fun/false). The previously-untried "exact tuple arity" axis is now directly observed in the consumers' pattern matches, not assumed.
- **Crash on stray input** — **upgraded `asserted → verified`.** Tree-wide grep proves `elixir_def.erl:293` is the sole writer of `{def,_}` rows, so the "foreign row shape" axis is closed by static evidence, not an external invariant.
- **Dead code** — **the one untried axis was tried (grep) and it surfaced a gap.** Promoted to low-confidence finding #1 (`no_module` orphaned); the remainder of the clause-reachability audit is now `verified`.
- **State/accumulator threading** — stays `asserted`. Reading `elixir_env:reset_read/2` confirmed the version counter is preserved (only `vars` is rewritten), which backs the Depends-on with evidence and removes the assumption. But the remaining axis is a runtime-differential (actual counter progression across a multi-default expansion), and per the pass rules a neighbour read cannot clear a runtime axis — I did not execute it, so it does not become `verified`.
- **Race condition** — stays `asserted`. The untried axis is same-module concurrent compilation, a property of `elixir_module`'s ownership model; confirming it needs a concurrency harness, not a read, so I leave it untried and the entry `asserted`.

### Dropped in re-review
None. Every prior ruled-out entry is preserved; three were strengthened to `verified` on read/grep evidence, and the `Dead code` dismissal was split — the reachability audit stays `verified` while its single previously-untried axis produced low-confidence finding #1.
