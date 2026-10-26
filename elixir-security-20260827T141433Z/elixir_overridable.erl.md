# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_overridable.erl
<!-- sha256: 856093002da1ab2d2d1b018b3624f97287c5f350c0812dfcf4b8da4dd1978e0e -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_overridable.erl
<!-- sha256: 856093002da1ab2d2d1b018b3624f97287c5f350c0812dfcf4b8da4dd1978e0e -->

_Re-reviewed on 2026-08-28 (hedge-resolution pass; flagged phrase: "outside this file")_

This pass targets the two remaining `asserted` ruled-out entries. For the `Macro.inspect_atom` entry I stopped reasoning about the out-of-tree implementation and instead **executed** it against adversarial atoms in the sandbox Elixir toolchain (`/opt/elixir/bin/elixir`), observing the actual escaping/return-type behaviour — the one axis previously on `Not examined` is now tried, so the entry is upgraded to `verified`. For the ETS check-then-act concurrency entry, the untried axis is *interleaving writers within one module's compilation*; that property is enforced by `elixir_code_server` and is not something I can exercise with an executable test, so per the rules it stays on `Not examined` and the entry stays `asserted` (the safe direction). No neighbour re-read was used to clear any axis.

## Summary

`elixir_overridable.erl` is a compile-time bookkeeper for overridable functions and `super`. Every sink is reachable only by compiling untrusted Elixir source — a path that already grants compile-time code execution via macros. Its one `list_to_atom` is source-proportional with no amplification, and its `io_lib:format` error strings are injection-safe (now confirmed including the `Macro.inspect_atom` argument path). No security findings.

Executed this pass:
- `Macro.inspect_atom(:remote_call, A)` on a battery of adversarial atoms (`:"~s~w~n.oops"`, `:"foo bar"`, `:"with\nnewline"`, `:"quote\"inside"`, `Foo.Bar`, `:"1abc"`, `:++`). Every result was a binary; special characters were escaped (`\n`, `\"`), format directives (`~s`, `~w`, `~n`) were left as inert literal bytes, and the value is consumed only as a `~ts` argument at line 131. No directive interpretation, no crash.

Neighbour reads carried forward from prior pass (reachability/term-origin, not used to clear any axis):
- `elixir_expand.erl:642` — `super/4` invoked during expansion. Reachability confirmed.
- `elixir_module.erl:493,496` — `store_not_overridden/1` invoked during `defmodule` compilation. Reachability confirmed.
- `elixir_module.erl:34-35` — `data_tables/1` = `ets:lookup_element(elixir_modules, Module, 2)`; tables keyed per-module (isolation confirmed).
- `elixir_def.erl:293` / `264-293` — the `{def,Tuple}` record shape and `store_definition/9` semantics; origin is compiler-internal.
- `elixir_aliases.erl:10-14` — `inspect/1` returns a binary, used only as a `~ts` argument.

## Security

No findings.

**Confidence: high** that there are no security findings in this file. The complete sink inventory is accounted for under **Ruled out**.

## Risks / caveats

- **Reachability is the load-bearing fact.** `super/4` is called only from `elixir_expand.erl:642` (during expansion) and `store_not_overridden/1` only from `elixir_module.erl:493,496` (during `defmodule` compilation). Any process reaching this file has already executed the source's macros — i.e., already holds arbitrary compile-time code execution. The parse-only public surface never reaches this module.
- **Out-of-`src` Elixir-side callers.** `record_overridable/3` and `overridable_for/2` are also called from the Elixir `Module` functions, which live outside the reviewable `src/` tree. Reachability was verified through the two in-folder callers named by the callgraph.
- **Internal `Def` shape (verified).** The hard destructures consume the `{def,Tuple}` record built at `elixir_def.erl:293`; compiler-constructed, not attacker-shaped.

### Ruled out

**`list_to_atom/1` — `name/2` (line 108), atom construction — Resource consumption (atom-table exhaustion).**\
Reason: creates one atom per `(function-name, override-count)` pair; `Count` only advances via a source-level `record_overridable` and each distinct atom costs proportional source, so no amplification, and the path is only reached after macros have executed (compile-time RCE already held).\
Axes examined: amplification (Count growth vs source size), interning/dedup of repeated `super` calls, integer radix/overflow of `Count`, reachability without compilation | Not examined: none.\
Basis: verified — read lines 81–108 and the `record_overridable` increment path (32–46); ran `list_to_atom(atom_to_list('~s~w~n.oops') ++ " (overridable 3)")` → single atom; reachability confirmed via `elixir_module.erl:493,496` and `elixir_expand.erl:642`.

**`io_lib:format/2` — `format_error/1` (lines 121, 124, 131), error message construction — Template / format-string injection.**\
Reason: all three format strings are constant literals; attacker-influenced values (`Name`, `Module`, `Joined`) are passed only as `~ts`/`~B` *arguments*, treated as data, not directives.\
Axes examined: `~` directives embedded in the atom argument, missing-argument crash, `~s` vs `~ts` on non-ASCII, arg count vs directive count | Not examined: none.\
Basis: verified — read lines 118–137; ran `io_lib:format` with `Name = '~s~w~n.oops'` and observed the literal echoed verbatim with no directive interpretation and no crash.

**`ets:lookup_element/insert_new/insert/update_element/lookup` — throughout (lines 16, 24, 32, 34, 36, 42, 53, 64, 66, 98) — Shared mutable state / ETS match-spec injection.**\
Reason: all operations use fixed-shape tuple keys with literal element positions; no `ets:select`/`ets:match` and no match specification anywhere.\
Axes examined: presence of any match-spec/`select`/`match` sink, key-shape controllability, element-position constants, cross-module table sharing | Not examined: none.\
Basis: verified — read the full file; no `select`/`match`/`fun2ms` present, every key is a literal-tagged tuple.

**ETS check-then-act sequences — `record_overridable/3` (32→36), `super/4`→`store/5` (53→96–98) — Concurrency (TOCTOU).**\
Reason: module compilation is single-process and serialized by `elixir_code_server`, so no second writer can interleave between the `insert_new`/`lookup` or `lookup`/`update_element` pairs on a module's tables.\
Axes examined: per-module table isolation (confirmed: `data_tables/1` returns tables keyed by `Module` in `elixir_modules`, `elixir_module.erl:34-35`) | Not examined: interleaving writers within one module's compilation (requires the `elixir_code_server` single-writer serialization guarantee; this property is not exercisable by an executable test I can construct, and reading the neighbour does not clear the axis).\
Basis: asserted — per-module table isolation is established, but the single-writer serialization it relies on is enforced by `elixir_code_server` and I ran no concurrency test that exercises interleaving, so the axis stays untried. Not a plausible finding: single-process compilation is a documented compiler invariant (standing threat model), and the worst case would be a self-inflicted crash of the compiling process.\
Depends on: elixir_code_server.erl — serialises `defmodule` compilation so a module's data tables have exactly one concurrent writer; elixir_module.erl — `data_tables/1` returns tables private to `Module` (read: line 34-35, confirmed per-module).

**Hard destructures of `Def`/`PreviousDef`/`Overridable` — `record_overridable/3` (37–38), `store/5` (82–83), `store_not_overridden/1` (70) — Deserialisation / validation (crash on malformed term).**\
Reason: these tuples are the `{def,Tuple}` record constructed internally at `elixir_def.erl:293`. The 6-tuple `{{def,{Name,Arity}}, Kind, Meta, File, Check, {Defaults,HasBody,LastDefaults}}` destructured here matches that construction exactly; producing a mismatched shape requires a caller to violate the `elixir_def` contract, and a mismatch only badmatches the caller's own compilation.\
Axes examined: origin of `Def` (internal vs external), consequence of mismatch (badmatch = caller crash only) | Not examined: none.\
Basis: verified — read `elixir_def.erl:264-293`; the record shape is built by `store_definition/9` at line 293. Origin is compiler-internal. **Upgraded from hedged assumption in first pass.**

**`elixir_def:store_definition/9` — `store/5` (99) — Code execution (defines a function in the target module).**\
Reason: it materialises a `defp`/`defmacrop` whose name/kind/clauses are copied from the already-recorded overridable definition; all inputs are internally derived and the result is a static function name written into the module's ETS tables, never a dynamic dispatch target.\
Axes examined: source of `FinalName`/`FinalKind`/`FinalClauses`, whether the name reaches dynamic `apply/3` | Not examined: none.\
Basis: verified — read `elixir_def.erl:264-293`; `store_definition/9` writes `Name`/`Kind`/`Clauses` as trusted compiler-generated input, with no `apply`/dynamic-call path. **Upgraded from hedged assumption in first pass.**

**`elixir_errors:file_error/4` — lines 44, 57, 74 — Template / error path.**\
Reason: routes to `format_error/1` in this module (injection-safe) and is a `no_return` diagnostic; the message data is function-name/kind, not a downstream interpreter payload.\
Axes examined: format-string safety of the reachable `format_error/1` clauses, argument provenance | Not examined: none.\
Basis: verified — `format_error/1` (this file) is the formatter and was tested above.

**`'Elixir.Enum':join/2`, `'Elixir.Macro':inspect_atom/2`, `integer_to_binary/1` — `format_error/1`/`format_fa/1` (130, 135–137) — Template / human-readable output.**\
Reason: these build a comma-joined list of available overridable `name/arity` strings for an error message; the result is concatenated into a constant `io_lib:format` template only as a `~ts` argument. `Macro.inspect_atom(:remote_call, Name)` returns a binary and escapes special characters; format directives inside the atom are inert data.\
Axes examined: whether the joined output re-enters a format string as a template (it enters as a `~ts` arg — confirmed at line 131), atom-inspection escaping/return-type of `Macro.inspect_atom(remote_call, _)` | Not examined: none.\
Basis: verified — **executed** `Macro.inspect_atom(:remote_call, A)` in the sandbox for adversarial atoms including `:"~s~w~n.oops"`, `:"with\nnewline"`, `:"quote\"inside"`, `Foo.Bar`, `:"1abc"`, `:++`. All returned binaries; embedded quotes/newlines were escaped (`\"`, `\n`) and format directives (`~s`, `~w`, `~n`) were preserved as literal bytes, then consumed as a `~ts` argument at line 131 — no directive interpretation, no crash. **Upgraded from hedged assumption in first pass** (was `asserted`: escaping of `Macro.inspect_atom` unverifiable because `lib/elixir/lib/macro.ex` is outside the reviewable tree). The axis is now cleared by execution rather than by reading the out-of-tree source.

### Did I verify this?

- **`list_to_atom`/`name/2`** — `verified`. Ran the atom build; reachability re-confirmed against `elixir_expand.erl:642` and `elixir_module.erl:493,496`. `Not examined: none`.
- **`io_lib:format`/`format_error/1`** — `verified`. Ran the format call with directive-laden input. `Not examined: none`.
- **ETS match-spec injection** — `verified`. Whole-file read confirms no `select`/`match`/`fun2ms`. `Not examined: none`.
- **`elixir_errors:file_error/4`** — `verified` via the tested `format_error/1` formatter. `Not examined: none`.
- **`Def`/`Overridable` destructures** — `verified`. Read `elixir_def.erl:293`: the destructured record is compiler-constructed. `Not examined: none`.
- **`elixir_def:store_definition/9`** — `verified`. Read `elixir_def.erl:264-293`: trusted compiler input, no dynamic dispatch. `Not examined: none`.
- **ETS check-then-act concurrency** — kept `asserted`. Per-module table isolation established via `elixir_module.erl:34-35`, but the single-writer serialization is in `elixir_code_server` and no interleaving test is constructable, so the axis stays untried. `Not examined: interleaving writers within one module's compilation.` Not a plausible finding.
- **`Enum.join`/`Macro.inspect_atom`** — **upgraded to `verified`.** Executed `Macro.inspect_atom(:remote_call, _)` against adversarial atoms and observed binary return + escaping; value is a `~ts` data argument. `Not examined: none`.

### Dropped in re-review

None. All prior ruled-out entries are preserved. The `Enum.join`/`Macro.inspect_atom` entry is upgraded (asserted → verified) on executed evidence; the ETS concurrency entry is retained as `asserted` (untried axis is not executable). No prior finding is dropped (the first pass reported no findings, and this pass reports none).
