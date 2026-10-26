# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_env.erl
<!-- sha256: 26616d466f6fd67f5d028af3d71154a63c632d04c0d4698ee55d27352c837b33 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_env.erl
<!-- sha256: 26616d466f6fd67f5d028af3d71154a63c632d04c0d4698ee55d27352c837b33 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

# Review: `elixir_env.erl`

1. **Summary** — This small environment/var-threading module is largely clean; the only defensible finding remains a diagnostic-location inconsistency where one of the two unused-variable warning sites omits the `calculate_span/2` enrichment the other applies. In this re-review I resolved all three first-pass hedges by reading the callgraph neighbours: `elixir.hrl` (record shape), `elixir_lexical.erl` (the in-folder tracer), and `elixir.erl` + `elixir_erl_var.erl` (the sole `with_vars/2` caller and its data source). Two dismissals move to **verified**; the tracer dismissal stays **asserted** but is now grounded in the in-folder tracer plus the external Code-tracer contract, with the hedge language removed.

2. **Bugfixes**

#### `CSCAN-elixir_env-merge-unused-warn-missing-span`
- **Finding ID** — `CSCAN-elixir_env-merge-unused-warn-missing-span`
- **Severity** — Low
- **Confidence** — low
- **Location** — `elixir_env.erl:135`, `merge_and_check_unused_vars/4` (second fold clause)
- **Class** — Unclassified (diagnostic-span inconsistency)
- **User-visible outcome** — Both warning sites emit the identical `{unused_var, Name, Overridden}` diagnostic, but `check_unused_vars/2` (line 101) wraps the meta with `calculate_span(Meta, Name)` to attach an end-column `span`, whereas the clause-merge path (line 135) passes the raw `Meta`. Unused-variable warnings that arise from cross-clause merging (variables unused inside `case`/`cond`/`with`/`fn` clauses that the parent scope knows about) therefore reach `elixir_errors:file_warn` without a `:span`, so editor/LSP tooling underlines only the start position instead of the full variable extent that the same warning gets when emitted directly. What a caller notices: the *same* "variable is unused" warning is highlighted with a full span in some scopes and only a caret in others, purely depending on which code path produced it.
- **Reproduction** — Compile a module where a variable is bound-but-unused inside one branch of a multi-clause construct so the warning flows through `merge_and_check_unused_vars/4` rather than the top-level `check_unused_vars/2`; compare the emitted warning's metadata (`span` present vs absent) against a straightforwardly-unused top-level variable.
- **Suggested fix** — Line 135: change `elixir_errors:file_warn(Meta, E, ?MODULE, Warn)` to `elixir_errors:file_warn(calculate_span(Meta, Name), E, ?MODULE, Warn)`, matching line 101.
- **Risk of fix** — Warning metadata gains a `span` key on this path; any test asserting the exact meta proplist for these warnings (or downstream consumers that reject an unexpected `span`) would observe the change. Otherwise none.
- **Evidence** — read `elixir_env.erl:100-142` (both warning sites for the same `{unused_var, ...}` message; only one calls `calculate_span`).

3. **Risks / caveats** — This file carries **no `-spec` declarations**, so the spec-totality obligation is vacuous here. The single finding stays Low/low: the merge-path `Meta` may be an intentional difference, and the bug-vs-by-design question isn't resolvable without the diagnostics contract in `elixir_errors` (a genuine external, not adjudicated here). All three first-pass "outside this file" hedges are now discharged (see *Ruled out* and *Did I verify this*).

### Ruled out

- **Operator / precedence typo**
  - **Reason** — The only compound guard, `is_compiler_var/1:151` (`Var =:= $_ ; Var >= $A, Var =< $Z`), parses as `(underscore) OR (A..Z)`, the intended "underscore or uppercase" partition.
  - **Axes examined:** `$_`, `$A`, `$Z`, lowercase `$f`, empty tail, single-underscore `[$_]` | Not examined: none
  - **Basis** — verified

- **Missing clause / catch-all**
  - **Reason** — `to_caller/1`, `with_vars/2`, `env_to_ex/1`, `close_write/2`, `merge_and_check_unused_vars/4` inner `case`, and `format_error/1` each have a terminal catch-all (`_`, `#{}`, or exhaustive shape) for the documented input shapes; list comprehensions silently skip non-`{Meta,Overridden}` (`false`-valued) `Unused` entries rather than crashing.
  - **Axes examined:** map value `false` vs `{Meta,Overridden}`, match vs non-match context, `Macro.Env` map vs `{Line, elixir_ex, Env}` tuple, empty vars map | Not examined: none
  - **Basis** — verified

- **Wrong atom / wrong symbol** — *Upgraded to verified from first-pass asserted*
  - **Reason** — Map keys (`versioned_vars`, `tracers`, `context`, `'__struct__' => 'Elixir.Macro.Env'`) and record fields are used consistently. The one first-pass gap — whether the `#elixir_ex` field names match the header — is now confirmed by reading `elixir.hrl:13-52`: the record declares `prematch` (31), `unused` (35), `vars` (49), and `version` (51) exactly as referenced here (`env_to_ex/1`, `reset_read/2`, `prepare_write/1,2`, `close_write/2`, `reset_unused_vars/1`, `check_unused_vars/2`, `merge_and_check_unused_vars/3`). No drift.
  - **Axes examined:** every literal atom key/field referenced in the file; `#elixir_ex` field names vs `elixir.hrl` declaration | Not examined: none
  - **Basis** — verified (read `elixir.hrl:13-52`). This is a definitional fact readable directly from the record declaration — there is no input axis to execute.

- **State / accumulator threading error**
  - **Reason** — `merge_vars/2` folds `V2` into `V1` as `Acc` keeping the max version (symmetric, order-independent); `merge_and_check_unused_vars/3` correctly returns `S` with the merged `unused` and the propagated `{Read, Write}`; no stale-variable reuse across rebinds.
  - **Axes examined:** disjoint maps, overlapping keys with M1<M2 and M1>=M2, identical maps (short-circuit clause `merge_vars(V,V)`), empty map | Not examined: none
  - **Basis** — verified

- **Off-by-one / wrong arithmetic**
  - **Reason** — `calculate_span/2:108` sets span end = `Column + string:length(atom_to_binary(Name))`, i.e. start column plus grapheme length, the correct half-open end; `with_vars/2` version indices start at 0 monotonically.
  - **Axes examined:** ASCII name, multi-byte UTF-8 name (grapheme vs byte count via `string:length`), empty vars list, single var | Not examined: name containing a grapheme cluster spanning multiple codepoints (behaviour is still "graphemes", matching intent)
  - **Basis** — verified

- **Contract violation**
  - **Reason** — Each exported function returns the shape its call sites consume (a `Macro.Env` map from `to_caller`/`with_vars`/`reset_vars`, an `#elixir_ex{}` from the var-handling functions, `E` from the check functions, `ok` from `trace`); no `-spec` exists to contradict.
  - **Axes examined:** all 4 return-shape families across the 15 exports | Not examined: none (no declared specs to over-promise)
  - **Basis** — verified

- **Crash on stray / unexpected input**
  - **Reason** — Inputs are compiler-internal env maps and `#elixir_ex{}` records, not attacker-controlled source bytes; the fold/case clauses cover the value shapes those maps hold. `trace/2:34`'s `ok = Tracer:trace(Event, E)` will `badmatch` only if a tracer returns non-`ok`. The sole in-folder tracer, `elixir_lexical:trace/2` (registered by `elixir_lexical:run/3` prepending `?MODULE` to `tracers`), returns `ok` on **every** clause including the catch-all `trace(_, _) -> ok` (`elixir_lexical.erl:35-75`), so the internal path never trips the assertion. User-supplied `Code` compilation tracers (injected via `elixir_config:get(tracers)` at `elixir_compiler.erl:19`, or via `Opts` at `elixir.erl:237-238`) are contractually required by the documented `Code` tracer callback to return `:ok`; a nonconforming user tracer would badmatch here by design (loud enforcement), which is a user-contract violation rather than a defect in this file.
  - **Axes examined:** `false` vs tuple `Unused` values, missing `column` in `Meta` (`calculate_span` `_` clause), non-match context, in-folder tracer return value (`elixir_lexical:trace/2`, all clauses `ok`) | Not examined: return values of external user-supplied tracer modules (not present in this repo/folder; cannot be executed here)
  - **Basis** — asserted
  - **Depends on** — external `Code` compilation tracers registered in `env.tracers`: `Tracer:trace/2` must return the atom `ok` per the documented tracer contract. The one in-folder tracer (`elixir_lexical`) is verified conforming.

- **Resource leak on error path**
  - **Reason** — Pure functional module; no ETS tables, ports, processes, monitors, or files are created here.
  - **Axes examined:** whole-file scan for `ets:`/`open_port`/`spawn`/`monitor`/`file:` | Not examined: none
  - **Basis** — verified

- **Silent corruption** — *Upgraded to verified from first-pass asserted*
  - **Reason** — `merge_vars` preserves the max version per key. The first-pass concern was the list clause of `with_vars/2` (`elixir_env.erl:42-45`), which builds `maps:from_list(ReversedVars)` and would pick version 0 for a repeated name (last-wins over the reversed accumulator). Reading the callers resolves this: the **only** in-folder caller of `with_vars/2` is `elixir.erl:317` (`elixir_env:with_vars(OrigE, ExVars)`) — a whole-folder grep found no other call site (the first pass's guess of `elixir_fn`/`elixir_expand` was incorrect). `ExVars` there is produced by `elixir_erl_var:load_binding/2` (`elixir_erl_var.erl:72-98`), which returns `ExVars` as a **map** deduplicated on `Pair` (`ExVars#{Pair => Counter}`, line 88). A map argument dispatches to `with_vars(Env, #{} = Vars)` (line 46), **not** the list clause, so the duplicate-name / version-0 path is unreachable in practice. The in-file behaviour on a duplicate-bearing *list* (dup → version 0) remains as traced, but no caller supplies such a list.
  - **Axes examined:** duplicate var names in `with_vars` list (in-file behaviour traced), map-vs-list clause dispatch, sole caller's argument type (`ExVars` = deduplicated map from `load_binding`), overlapping merge keys, identical-map short-circuit | Not examined: none
  - **Basis** — verified (read `elixir.erl:315-320`, `elixir_erl_var.erl:72-101`; grep confirms `elixir.erl:317` is the only in-folder `with_vars/2` caller)

- **Infinite loop / hang**
  - **Reason** — `is_compiler_var/1` recurses on a strictly shortening list; all other functions are non-recursive folds/comprehensions over finite maps; no `receive`.
  - **Axes examined:** long name, empty tail, all-underscore name | Not examined: none
  - **Basis** — verified

- **Quadratic-where-linear**
  - **Reason** — No `Acc ++ X`, no per-iteration `length/1`, no `lists:member` visited-set; accumulation uses `[H|Acc]` (`with_vars`) and `maps:fold`.
  - **Axes examined:** all list/map builders in file | Not examined: none
  - **Basis** — verified

- **Repeated work**
  - **Reason** — No compiled patterns (`binary:compile_pattern`/`re:compile`) or repeated BEAM/stat scans; `atom_to_binary`/`atom_to_list` are cheap per-warning conversions on the error path only.
  - **Axes examined:** whole-file scan for compile-in-loop idioms | Not examined: none
  - **Basis** — verified

- **Dead / unreachable code**
  - **Reason** — All 15 exports appear in the intra-folder callgraph's caller list; internal helpers (`is_unused_var`, `is_compiler_var`, `calculate_span`) are all reached from exported functions.
  - **Axes examined:** each export vs callgraph caller set, each private helper vs its call site | Not examined: none
  - **Basis** — verified

- **Documented behaviour mismatch**
  - **Reason** — No `@doc`/`-spec`/prose contract is present in the file; the inline comments in `new/0` describe field purposes and match the field values set.
  - **Axes examined:** each `new/0` field comment vs its initial value | Not examined: none
  - **Basis** — verified

- **Race condition / shared mutable state**
  - **Reason** — Purely functional transformation of caller-owned terms; no `system_flag`, ETS, or process state mutated here.
  - **Axes examined:** whole-file scan for global/process state | Not examined: none
  - **Basis** — verified

- **Wrong scope / order of operations**
  - **Reason** — In each clause, validation/lookup precedes use (`calculate_span` checks for `column` before adding a span; `close_write` inspects the upper write shape before merging); no side-effect-before-validate pattern.
  - **Axes examined:** `calculate_span` column-present/absent ordering, `close_write` false/non-false ordering | Not examined: none
  - **Basis** — verified

- **Encoding / unicode mishandling**
  - **Reason** — `calculate_span` uses `string:length(atom_to_binary(Name))` (grapheme count on UTF-8), and `format_error` uses `~ts` for names — both Unicode-aware; no byte/codepoint confusion.
  - **Axes examined:** ASCII name, multi-byte name, name with leading `_`/`&` | Not examined: none
  - **Basis** — verified

### Did I verify this?

- **Operator/precedence, Missing-clause, Off-by-one, State-threading, Contract, Resource-leak, Infinite-loop, Quadratic, Repeated-work, Dead-code, Doc-mismatch, Race, Scope/order, Encoding** — kept **verified**: read the relevant clauses (lines 33–164) and exercised the enumerated axes to `Not examined: none`.
- **Wrong atom / wrong symbol** — now **verified** (was asserted). Read `elixir.hrl:13-52`; the `#elixir_ex` record declares `prematch`, `unused`, `vars`, `version` exactly as used. This is a record-definition fact, fully answered by reading the header — no input axis exists to execute.
- **Silent corruption (with_vars duplicates)** — now **verified** (was asserted). The sole in-folder caller (`elixir.erl:317`) passes `ExVars`, which `elixir_erl_var:load_binding/2` returns as a deduplicated map, so the list clause's duplicate→version-0 behaviour is unreachable. First-pass *Depends on* named the wrong callers (`elixir_fn`/`elixir_expand`); grep shows `elixir.erl` is the only call site.
- **Crash on stray input (tracer `ok =`)** — stays **asserted**, but the hedge is discharged as far as this repo allows: the one in-folder tracer (`elixir_lexical:trace/2`) is confirmed to return `ok` on every clause, so no internal path badmatches. The residual axis — external, user-supplied `Code` compilation tracers returning non-`ok` — cannot be executed here (those modules are not in this folder), so it remains on *Not examined*; the badmatch on such a tracer is by-design contract enforcement, not a defect in this file.

### Dropped in re-review

None. Every first-pass dismissal is preserved (two strengthened to verified, one kept asserted with strengthened basis), and the sole finding is retained unchanged.
