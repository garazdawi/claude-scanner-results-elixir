# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl.erl
<!-- sha256: 8aba02532d3269c6318e5d10842a61ee94035a3a18085ac31a2cf0552742805e -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl.erl
<!-- sha256: 8aba02532d3269c6318e5d10842a61ee94035a3a18085ac31a2cf0552742805e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

# Bug-hunting review — `elixir_erl.erl`

## Summary
The Erlang backend is clean on its documented public surface; no correctness, crash, leak, or contract-violation bug survived tracing from a public entry point. On re-review I read the sole cross-file producer/owner (`elixir_module.erl`) for the three previously-`asserted` dismissals and confirmed each dependency holds with concrete evidence; all three now carry `Not examined: none`.

## Bugfixes

No findings.

The public API is `elixir_to_erl/1,2`, `definition_to_anonymous/5`, `compile/2`, `consolidate/4`, `get_ann/1`, `debug_info/4`, `scope/2`, `checker_chunk/2`, `checker_version/0`, `format_error/1`. The only entry point that consumes broadly-shaped, caller-supplied data is `elixir_to_erl/1,2` (arbitrary Elixir literal terms); every other export is invoked by in-folder compiler drivers (`elixir_compiler`, `elixir_def`, `elixir_module`, `elixir_erl_pass`) with compiler-internal, already-validated maps. I traced each and found no defect that produces an outcome a caller would observe that isn't intended.

Two behaviours I looked hard at and cleared:

- **Signed-zero float rewriting** (`elixir_to_erl/2`, lines 86–93). Guard `is_float(Tree), Tree == 0.0` catches both `+0.0` and `-0.0` (numeric `==`). The big-endian 64-bit encoding `<<Tree/float>>` places the sign bit first, so `<<1:1,_:63>>` correctly discriminates `-0.0` → `'-'` and `+0.0` → `'+'`, each emitted as `op` over the `{float,Ann,0.0}` literal. Matches the stated intent ("rewrite 0.0 as AST for +0.0 in matches"). No bug.
- **`usort_callbacks/1` comparator** (lines 396–403). `LineComparator` returns a total `=<`-style order (equal callbacks by line, else by callback term), valid for `lists:sort/2`; `lists:uniq/2` then keeps the earliest-line duplicate as the comment promises. No ordering/precedence defect.

## Risks / caveats
- I did not execute an Erlang toolchain for the float/bitstring/`erl_anno` paths; they were traced by reading. Confidence on the signed-zero and bitstring paths is high from the code alone. None of these are cross-file, so no dismissal below rests on them.
- The three cross-file dismissals previously marked `asserted` are now backed by reading `elixir_module.erl` (the callgraph producer of the compiler `Map` and the owner of the Set/Bag ETS tables). Specific lines cited per entry. The remaining depends-on lines for `elixir_erl_pass:translate_args/3` and `elixir_errors:file_error/4` are contract assumptions on those neighbours; they are not among the flagged hedges and are left as noted context, not as ruled-out Basis.

### Ruled out

**Operator / precedence typo**
- Reason: Guards read correctly under explicit parentheses — `is_atom(Name) and is_integer(Arity)` (line 434), `Kind == callback; Kind == macrocallback` (line 425), float guard `is_float(Tree), Tree == 0.0` (line 86); none partition to empty or universal sets.
- Axes examined: `and` vs `;` connectives, `==` vs `=:=` on floats incl. -0.0, guard on non-float | Not examined: none.
- Basis: verified

**Missing clause / catch-all**
- Reason: `elixir_to_erl/2` ends in a `error(badarg,...)` catch-all (line 118) and `debug_info/4` in `{error, unknown_format}` (line 43); the only unguarded `case` is `dynamic_form`'s `anno`/`line` selection (lines 187–190). I traced every producer of that `Map`. The live compiler path builds `ModuleMap` in `elixir_module:compile/7` with `anno => Anno` present unconditionally (`elixir_module.erl:218`), and this same map is what `elixir_erl:compile/2` (`elixir_module.erl:232`) forwards to `dynamic_form`. The `debug_info(erlang_v1|core_v1,...)` and `consolidate/4` callers receive the identical map (round-tripped through the stored `{elixir_v1, Map, Specs}` debug chunk written at `load_form`/`debug_opts`, line 530). The `line` branch exists only to accept legacy pre-v1.22 stored chunks (see the TODO at line 186). A map carrying *neither* key is therefore not producible by any in-tree caller — the case is total across every real input.
- Axes examined: tuple/list/map/atom/int/float/binary/bitstring/fun/pid/port/ref/improper-list into elixir_to_erl; unknown debug_info format; compiler `Map` with `anno` (live path) vs `line` (legacy debug chunk) into dynamic_form | Not examined: none.
- Basis: verified — cross-file producer read: `elixir_module.erl:215-229` (map literal, `anno => Anno` at :218) and forward at :232.

**Wrong atom / wrong symbol**
- Reason: `__info__` clause set (lines 329–339) matches `AllowedAttrs` (line 315) one-to-one — module, functions, macros, struct, exports_md5, attributes, compile, md5, deprecated.
- Axes examined: each __info__ key vs allowed-args union, record field `#elixir_erl.caller`/`.expand_captures` | Not examined: none.
- Basis: verified

**State / accumulator threading error**
- Reason: `translate_clause` threads `TS` (post-translate scope) only to read `TS#elixir_erl.caller`, using `TClause`/`element(3/5,...)` consistently; `split_definition` accumulators (`Def`/`Defmacro`/`Macros`/`Exports`/`Functions`) are passed forward intact on every clause.
- Axes examined: macro vs non-macro clause, caller true/false, def/defp/defmacro/defmacrop, unreachable skip | Not examined: none.
- Basis: verified

**Off-by-one / wrong arithmetic**
- Reason: Only arithmetic is macro arity `Arity + 1` (lines 264, 485) paired with the prepended `_@CALLER`/`term` type — consistent on both the function-head and spec sides.
- Axes examined: macro arity bump in translate_definition vs spec_for_macro vs callspecs_form Key | Not examined: none.
- Basis: verified

**Contract violation**
- Reason: `debug_info/4` returns only documented `{ok,_}`/`{error,_}` shapes; `checker_chunk/2` returns `[]` or a one-chunk list; `elixir_to_erl` always returns a well-formed abstract-format tuple or raises `badarg`.
- Axes examined: debug_info all four format atoms + failure paths, checker_chunk nil vs contents | Not examined: none.
- Basis: verified

**Crash on stray / unexpected input**
- Reason: `elixir_to_erl` is total over all Erlang terms — every non-serializable term (funs with env, and the final catch-all) routes to an intentional `error(badarg,...)`, which is the documented behaviour, not a defect.
- Axes examined: external/local fun, fun with non-empty env, pid/port/ref, improper list, non-byte bitstring, empty binary, empty map, empty list | Not examined: none.
- Basis: verified

**Resource leak on error path**
- Reason: This module allocates nothing releasable. Every `ets:*` call here is a read (`ets:lookup`/`ets:lookup_element`), and the tables are created and destroyed entirely in the owner: `elixir_module:build/4` creates both with `ets:new(Module, [set, public])` (`elixir_module.erl:411`) and `ets:new(Module, [duplicate_bag, public])` (:425), and the `after` clause of `elixir_module:compile/7` deletes both — `ets:delete(DataSet)` / `ets:delete(DataBag)` (`elixir_module.erl:266-267`) — on every exit, success or crash. `elixir_erl` never opens files, spawns, monitors, or allocates ETS/processes; `term_to_binary` holds no resource. There is no error path in this module that could strand anything.
- Axes examined: every ets:* call site in this file (all read-only); term_to_binary (no resource); owner create/delete lifecycle in the neighbour | Not examined: none.
- Basis: verified — cross-file owner read: `elixir_module.erl:411,425` (create) and `:264-268` (`after` → `ets:delete` both tables).

**Silent corruption**
- Reason: Map assoc list is sorted deterministically (line 78); binary→charlist conversion documents its latin1/UTF-8 intent (lines 96–101); no accumulator is dropped on an empty-collection clause (`[]`→nil, `<<>>`→empty bin, `#{}`→empty map all explicit).
- Axes examined: empty map/list/binary, map key ordering determinism, bitstring trailing fragment | Not examined: none.
- Basis: verified

**Infinite loop / hang**
- Reason: No `receive`, no `gen_server` handlers, no retry loops in this module; all recursion (`elixir_to_erl_cons`, `split_definition`, folds) strictly consumes its list.
- Axes examined: cons recursion on improper list, split_definition list consumption | Not examined: none.
- Basis: verified

**Quadratic-where-linear**
- Reason: `add_definition` builds `{Head, Tail}` with prepends and a single `Head ++ Tail` at the terminal clause (lines 219, 246–258); `attributes_form` uses `foldr`; no `Acc ++ X` inside a loop.
- Axes examined: split_definition accumulation, functions body concat, docs list comprehensions | Not examined: none.
- Basis: verified

**Repeated work**
- Reason: No pattern/regex compilation; `dynamic_form` is computed once per `compile`/`consolidate`/`debug_info` call.
- Axes examined: per-call recompilation candidates (none present) | Not examined: none.
- Basis: verified

**Dead / unreachable code**
- Reason: Every `elixir_to_erl` clause is reachable via a distinct term type; both `debug_info` `elixir_v1` clauses (`none` vs `{elixir_v1,...}`) are reachable from beam_lib depending on stored chunk.
- Axes examined: elixir_to_erl clause reachability by type, debug_info format dispatch | Not examined: none.
- Basis: verified

**Documented behaviour mismatch**
- Reason: The module's contracts are the leading comments ("Converts Elixir quoted literals to Erlang AST", "debug_info callback", etc.); each function's behaviour matches its comment, and there are no `-spec` declarations to contradict.
- Axes examined: each export's comment vs behaviour (elixir_to_erl, debug_info, checker_chunk, get_ann, scope) | Not examined: none.
- Basis: verified

**Race condition / shared mutable state**
- Reason: The Set/Bag tables are **per-module, not shared across compiles**: `elixir_module:build/4` calls `ets:new(Module, ...)` fresh for each module definition (`elixir_module.erl:411,425`), so two concurrent compiles operate on distinct tables — the "concurrent compiles sharing the Set table" axis is structurally impossible. Within a single compile, all of `elixir_erl`'s reads run inside the one process spawned at `elixir_erl_compiler:spawn` (`elixir_module.erl:169-236`), and that process calls `make_readonly(Module)` (`elixir_module.erl:188`, `ets:update_element` setting mode `readonly`) *before* invoking `elixir_erl:compile` (:232). The parallel checker that could touch the map is spawned only *after* `elixir_erl:compile` returns (:234). So no writer is concurrent with `elixir_erl`'s reads. This module itself performs no `system_flag` and no read-then-write on any shared table.
- Axes examined: ets read-only usage in this file; per-module table creation vs a shared table; write/readonly ordering relative to elixir_erl:compile; parallel-checker spawn ordering | Not examined: none.
- Basis: verified — cross-file read: `elixir_module.erl:411,425` (per-module `ets:new`), `:188` (`make_readonly` before), `:232` (elixir_erl:compile), `:234` (checker spawned after).

**Wrong scope / order of operations**
- Reason: `add_definition` computes `FileMeta` and reads original `Meta` before use; `compile/2` merges env compiler options into `compile_opts` before `dynamic_form` reads them (lines 153–169); no side-effect precedes its input validation.
- Axes examined: env-option merge ordering, file-attr meta capture ordering | Not examined: none.
- Basis: verified

**Encoding / unicode mishandling**
- Reason: `binary_to_list(Tree)` (line 101) is deliberate and documented — Erlang treats binary-embedded strings as latin1 so bytes pass through unchanged into a `{string,...}` bin_element; UTF-8 bytes are preserved, not re-decoded.
- Axes examined: multibyte UTF-8 binary, empty binary, non-byte bitstring | Not examined: none.
- Basis: verified

### Did I verify this?

- **Missing clause / catch-all** — upgraded `asserted` → `verified`. I read the producer of the `dynamic_form` map: `elixir_module:compile/7` builds `ModuleMap` with `anno => Anno` present unconditionally (`elixir_module.erl:215-229`, key at :218) and forwards that exact map to `elixir_erl:compile/2` at :232; the `debug_info`/`consolidate` callers reuse the same map via the stored debug chunk, and the `line` branch only accepts legacy chunks. A map lacking both keys is not producible, so `Not examined` is now `none`. This is a structural map-shape invariant, not a differential-guard axis, so reading the sole producer is conclusive; no execution would tell me more.
- **Resource leak on error path** — upgraded `asserted` → `verified`. The ETS lifecycle lives entirely in `elixir_module` (`ets:new` at :411/:425, `ets:delete` for both tables in the `after` clause at :266-267, on every exit). `elixir_erl` only reads. Ownership/lifecycle is fully determined by reading the owner; there is no axis execution could clear.
- **Race condition / shared mutable state** — upgraded `asserted` → `verified`. Tables are created per-module (`ets:new(Module, …)` per `build/4`), so distinct compiles never share a table; and within a compile the table is made `readonly` (`make_readonly`, :188) before `elixir_erl:compile` (:232) reads it, with the parallel checker spawned only afterward (:234). The single untried axis (concurrent compiles sharing the Set table) is refuted structurally by per-module table creation, so `Not examined` is now `none`.
- All entries already marked `verified` in the first pass were re-checked and remain in-file, single-datatype-axis, pure logic with no cross-file dependency; re-reading substantiates them unchanged.

### Dropped in re-review

None. Every entry from the prior report is preserved; the only changes are the three cross-file dismissals moving from `asserted` to `verified` with concrete `elixir_module.erl` line citations, and no prior finding existed to drop.
