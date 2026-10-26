# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_overridable.erl
<!-- sha256: 856093002da1ab2d2d1b018b3624f97287c5f350c0812dfcf4b8da4dd1978e0e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T06:24:02Z_

# Review: `elixir_overridable.erl`

1. **Summary** — This is a small, internally-consistent compiler helper module (overridable-function bookkeeping over the module ETS tables); I found no correctness, crash, or contract bugs reachable from its documented in-folder API.

2. **Bugfixes**

No findings.

The module has no `-spec` declarations and no exported functions with non-builtin spec types, so the spec-totality checklist is empty. I traced each export:

- `overridables_for/1` — `ets:lookup_element(Bag, overridables, 2)` on a bag returns the list of recorded tuples; `try/catch _:_ -> []` correctly absorbs the `badarg` when the key/table is absent. Return shape (`[Tuple]`) matches all callers.
- `overridable_for/2` — returns the full 4-tuple row `{{overridable,Tuple},Count,Def,Overridden}` or `not_overridable`; both shapes are handled by callers (`elixir_expand`).
- `record_overridable/3` — `insert_new` / increment path is race-free for the single compiler process; `Count+1` is the only arithmetic and is correct; `{{_,Kind,Meta,File,_,_},_} = Def` matches the 6-element def row wrapped with clauses; Bag and Set stay in sync because the Bag insert (line 34) happens only on a successful `insert_new`.
- `super/4` — missing-key path emits `{no_super,...}` via `file_error` (env `E` is accepted by `elixir_errors:file_error/4`, which handles both `binary()` and `#{file:=binary()}`).
- `store_not_overridden/1` — `[Overridable] = ets:lookup(...)` cannot `badmatch` because every Bag `overridables` entry corresponds to exactly one Set row (kept 1:1 by `record_overridable`); `element(3, Overridable)` correctly selects `Def`.
- `store/5`, `name/2`, `is_valid_kind/2`, `is_macro/1`, `format_error/1`, `format_fa/1` — traced below in Ruled out.

3. **Risks / caveats** — My dismissals of the ETS row shapes (`{def,Tuple}` is a 6-tuple; the overridable row is `{key,Count,Def,Overridden}` with `Def = {DefRow6,Clauses}`) and of `elixir_def:store_definition/9`'s argument order depend on `elixir_def`/`elixir_module` producing exactly those shapes; I confirmed `elixir_errors:file_error/4` accepts both a binary file and an env map by reading `elixir_errors.erl:264-267`, but did not read `elixir_def.erl`/`elixir_module.erl` in full. If the def-row arity or the overridable-row layout drifts in those modules, several `badmatch`-free claims here would flip; the `?overridden_pos 4` constant is the most fragile coupling. No Erlang runtime was available, so all tracing is static.

### Ruled out

**Operator / precedence typo**
Reason: The only compound guard, `Kind == defmacro; Kind == defmacrop` (line 90), is an `;`/orelse partition that correctly matches the two macro kinds; `==` on atoms is equivalent to `=:=`.
Axes examined: Kind ∈ {defmacro, defmacrop, def, defp}, Hidden ∈ {true,false} | Not examined: none
Basis: verified

**Missing clause / catch-all**
Reason: `case Hidden` (line 87) covers `false`/`true`(macro)/`true`(other) exhaustively over a boolean; `case Overridden` (line 96) covers `false`/`true`; `is_macro/1` has a `_` fallthrough; `format_error/1` handles both emitted error atoms (`bad_kind`, `no_super`).
Axes examined: Hidden true/false, Overridden true/false, Kind macro/non-macro, error atom bad_kind/no_super | Not examined: none
Basis: verified

**Wrong atom / wrong symbol**
Reason: `?overridden_pos` is 4, matching the `Overridden` field (pos 1 key, 2 Count, 3 Def, 4 Overridden) used consistently by `store/5`'s `{_,Count,Def,Overridden}` destructure and `overridable_for`; ETS keys `{overridable,Tuple}`/`overridables`/`{def,Tuple}` are used consistently across read and write sites.
Axes examined: each field position 1–4 of the overridable row, both table keys | Not examined: none (row layout produced by `elixir_module`/`elixir_def` assumed — see Depends on)
Basis: asserted
Depends on: `elixir_module:data_tables/1` returns `{Set,Bag}` and `elixir_def` stores the overridable row as `{{overridable,Tuple},Count,Def,false}` with `Def = {DefRow,Clauses}`.

**State / accumulator threading error**
Reason: No mutable accumulator threading; `Count`/`Overridden` are read from ETS and written back atomically via `ets:insert`/`update_element`; the incremented count uses the freshly-looked-up `Count`.
Axes examined: first-record vs re-record (insert_new true/false), super-once vs super-twice (Overridden false→true) | Not examined: none
Basis: verified

**Off-by-one / wrong arithmetic**
Reason: Only arithmetic is `Count + 1` for the redefinition counter, which is correct and monotonic; `name/2` uses the current `Count` verbatim.
Axes examined: Count = 1, Count after multiple redefinitions | Not examined: none
Basis: verified

**Contract violation**
Reason: All public functions return the single shape their callers expect (`[Tuple]`, row-or-`not_overridable`, `ok`, `{FinalKind,FinalName,Meta}`, chardata); no divergent return shapes across clauses.
Axes examined: each export's return branches | Not examined: cross-file caller expectations for `store/5`'s tuple (out of file)
Basis: asserted
Depends on: `elixir_expand` consumes `{FinalKind,FinalName,Meta}` from the `super` path.

**Crash on stray / unexpected input**
Reason: Inputs are compiler-internal tuples produced by `elixir_def`/`elixir_module`, not attacker-controlled source; the sole external-shape dependency (`ets:lookup_element`) is guarded by `try/catch`.
Axes examined: absent ETS key (caught), empty vs single lookup result | Not examined: malformed def-row arity (would require `elixir_def` contract break)
Basis: asserted
Depends on: `elixir_def` def rows are 6-tuples and overridable `Def` fields are `{DefRow,Clauses}`.

**Resource leak on error path**
Reason: No file handles, processes, monitors, or ETS tables are created here; the module only reads/updates tables owned by `elixir_module`.
Axes examined: error path via `file_error` (no_super, bad_kind) | Not examined: none
Basis: verified

**Silent corruption**
Reason: `store/5` stores the hidden/original def exactly once (guarded by the `Overridden` flag flipped via `update_element`), so repeated `super` calls neither double-store nor drop the def; names are deterministic from `Count`.
Axes examined: Overridden false (store) vs true (skip), Hidden true (renamed defp/defmacrop) vs false (original name) | Not examined: none
Basis: verified

**Infinite loop / hang**
Reason: No recursion, no `receive`, no gen_server loop in this module; `lists:foreach` iterates a finite ETS result.
Axes examined: empty overridables list, non-empty list | Not examined: none
Basis: verified

**Quadratic-where-linear / Repeated work**
Reason: `store_not_overridden/1` does one `ets:lookup` per overridable inside a single `foreach`; no `++` accumulation, no per-iteration compile of patterns; `format_error` builds `Bins` with a comprehension then a single `Enum.join`.
Axes examined: N overridables → N lookups (linear), join over list | Not examined: none
Basis: verified

**Dead / unreachable code**
Reason: Every clause is reachable from `elixir_expand`/`elixir_module` entry points; both `format_error` clauses correspond to errors this module actually raises.
Axes examined: each function head and case branch | Not examined: none
Basis: verified

**Documented behaviour mismatch**
Reason: The only prose is the module header comment ("defining overridable functions and handling super"); the code matches that description and there are no `-spec`/`@doc` claims to contradict.
Axes examined: module comment vs behaviour | Not examined: none
Basis: verified

**Race condition / shared mutable state**
Reason: These ETS tables are owned and mutated by the single compiler process per module; `insert_new`/`update_element` are atomic; no concurrent writer is documented.
Axes examined: single-process compile assumption | Not examined: concurrent compilation of the same module (not a documented mode)
Basis: asserted
Depends on: `elixir_module` compiles a given module in one process owning `{Set,Bag}`.

**Wrong scope / order of operations**
Reason: `record_overridable` performs the `is_valid_kind` check before inserting the incremented row; `store` flips `Overridden` before `store_definition` but both are idempotent w.r.t. the flag; `bad_kind` uses the new def's `Meta`/`File`, which is the correct error location.
Axes examined: valid vs invalid kind ordering, first vs subsequent super | Not examined: none
Basis: verified

**Encoding / unicode mishandling**
Reason: `name/2` round-trips atom→list→atom purely for a synthetic suffix; `format_fa` uses `Macro.inspect_atom/2` and binary concat, and `format_error` uses `~ts` (unicode) format specifiers — no byte/codepoint confusion.
Axes examined: unicode atom name into `~ts`, atom_to_list round-trip | Not examined: none
Basis: verified

**Unclassified**
Reason: No file-specific one-off defect surfaced after tracing all exports and their cross-file couplings.
Axes examined: full-file trace | Not examined: none
Basis: verified

### Did I verify this?

- The `verified` entries above (operator typo, missing clause, threading, arithmetic, resource leak, silent corruption, hang, quadratic, dead code, doc mismatch, order-of-ops, encoding, unclassified) were traced with every listed axis tried against the in-file logic; their `Not examined` lists read `none`, so they stand as verified.
- **Wrong atom / wrong symbol** stays `asserted`: I verified in-file consistency of `?overridden_pos` and the field positions, but the ground truth of the row layout lives in `elixir_module`/`elixir_def`, which I did not read. Kept `Depends on:` naming that contract.
- **Contract violation** and **Crash on stray input** stay `asserted`: the untried axes (caller expectations for `store/5`'s return; a malformed def-row arity) are only reachable by reading `elixir_expand`/`elixir_def`, out of this file's scope. `Depends on:` lines name where verification must occur.
- **Race condition** stays `asserted`: the single-compiler-process invariant is an `elixir_module` property, not observable in this file; `Depends on:` records it.
- I confirmed the one cross-file claim I could reach locally — `elixir_errors:file_error/4` accepting both a binary file and an env map — by reading `elixir_errors.erl:264-267`, so the `super/4` vs `record_overridable/3` second-argument asymmetry is not a bug.

Evidence:
- read /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_overridable.erl:1-138
- read /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_errors.erl:264-289 (`file_error/4` accepts `binary() | #{file := binary()}`)

