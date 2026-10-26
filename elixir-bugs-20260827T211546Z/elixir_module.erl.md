# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_module.erl
<!-- sha256: bd1e730a8bef579e19f5787787564eda1bacbb109403b6d27982d4f1432e247f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T06:23:04Z_

# Bug-hunting review — `elixir/lib/elixir/src/elixir_module.erl`

## Summary
After tracing every exported entry point against the ETS-table and env-threading contracts it depends on, I found no defect reachable through the module's documented public API.

## Bugfixes

No findings.

The exported surface (`compile/6`, `expand_callback/6`, `file/1`, `data_tables/1`, `is_open/1`, `mode/1`, `delete_definition_attributes/6`, `write_cache/3`, `read_cache/2`, `next_counter/1`, `taint/1`, `cache_env/1`, `get_cached_env/1`, `exports_md5/3`, `compiler_modules/0`, `format_error/1`) was walked clause by clause. The areas most likely to hide bugs were checked and dismissed:

- **`validate_module_name/1` (133-140)** — guard `Module == nil; is_boolean(Module); not is_atom(Module)` correctly rejects `nil`/`true`/`false`/non-atoms via disjunction, and the `$/`/`$\\` scan rejects path-injecting module names before `beam_location/1` builds a filename from the charlist. Correct and reached before any side effect.
- **`taint/1` (73-80)** inserts `{{elixir, taint}}` (arity-1 tuple) whose set-table key is `{elixir, taint}`, which is exactly what `compile_error_if_tainted/2` (271-275) tests with `ets:member/2`. Consistent.
- **`compile/7` catch/after (248-269)** purges/deletes the module on both the `undef` and generic paths and always tears down `DataSet`/`DataBag` and calls `undefmodule` in `after`, and restores `put_compiler_modules(CompilerModules)` — no ETS/monitor leak on the error path.
- **`mode/1` (40-45)** catches `_:badarg` → `closed`, matching the "table may be gone" contract.

## Risks / caveats
- This module is compiler-internal; its "documented public API" is the callable surface listed in the callgraph slice, and nearly all of it assumes the two per-module ETS tables built by `build/4` exist. Helpers like `file/1`, `data_tables/1`, `read_cache/2` will `badarg` if called for a module that was never opened — but that is the documented contract (`is_open/1`/`mode/1` are the guarded accessors), not a bug.
- Several dismissals below depend on neighbour-module behaviour (`elixir_code_server`, `elixir_env`, `elixir_erl_compiler`, `elixir_config`); those dependencies are named in `Depends on:` lines for the cross-file pass to confirm.

### Ruled out

**Operator / precedence typo**
Reason: The only compound guard, `validate_module_name/1:133`, is a pure disjunction whose three alternatives partition exactly the invalid-name space; no `,`/`;` mixing and no `==`-vs-`=:=` hazard (atoms compared).
Axes examined: `nil`, `true`, `false`, non-atom (integer/tuple), ordinary atom, atom containing `/` or `\` | Not examined: none
Basis: verified

**Missing clause / catch-all**
Reason: The `case`/function heads that face external data (`validate_compile_opt`, `lookup_attribute`, `beam_location`, stacktrace match in the `undef` handler) all carry fall-through clauses; the `undef` handler's third clause re-raises rather than crashing.
Axes examined: unknown compile opt (falls to `[Opt]`), non-accumulate/unset/value attribute, empty ETS lookup, stacktrace whose head is not this module | Not examined: none
Basis: verified

**Wrong atom / wrong symbol**
Reason: Attribute keys, ETS positions, and taint/cache key tuples used here match their producers/consumers (`?counter_attr`, `?cache_key`, `{elixir, taint}`, `{cache, Key}`), verified against the `build/4` inserts.
Axes examined: taint key insert vs member test, cache key insert vs lookup, persisted-attribute list vs `lookup_attribute` | Not examined: none
Basis: verified

**State / accumulator threading error**
Reason: `compile/7` threads `ModuleE`/`CallbackE`/`E` distinctly and uses each for its documented trace/callback; `eval_form/7` returns `{Value, prunedEnv, EC}` with the pruned env, not a stale one.
Axes examined: env passed to `after_compile` vs `on_module` trace, `Prune=true`/`false` branches of `maybe_prune_versioned_vars` | Not examined: none
Basis: verified

**Off-by-one / wrong arithmetic**
Reason: The only arithmetic is `NewKey = PrevKey + 1` (monotonic cache-env id) and `PruneBefore = length(Vars)` used as a `<` threshold; both are consistent with their reads.
Axes examined: empty `Vars` (PruneBefore=0), repeated `cache_env` calls (key increments), first call (PrevKey=0 from `build`) | Not examined: none
Basis: verified

**Contract violation**
Reason: `next_counter/1` returns a bare `unique_integer()` for `nil`/ETS-failure and `{Module, N}` otherwise; both are used only as opaque unique hygiene contexts and the two shapes are the intended, long-standing contract, not a drift.
Axes examined: `nil` arg, valid Module, Module with missing/deleted table (catch path) | Not examined: none
Basis: verified

**Crash on stray / unexpected input**
Reason: The public compile path validates the module name first and every attribute-validation helper (`validate_on_load_attribute`, `validate_dialyzer_attribute`, `validate_nifs_attribute`, `validate_definition`) has a `false`/catch-all clause; `bag_lookup_element/3` swallows `badarg` to `[]`.
Axes examined: missing on_load/dialyzer/nifs attribute (`false`), malformed dialyzer via `lists:flatten([Dialyzer])`, absent ETS bag entries | Not examined: attribute values whose internal shape is guaranteed by `Module.put_attribute` upstream
Basis: asserted
Depends on: `Elixir.Module`/`elixir_def` — accumulated attribute values inserted into the bag have the `{Key,Funs}`/`{Name,Arity}` shapes the validators pattern-match; if an upstream inserts a differently-shaped term a `function_clause` could surface, but that path is not reachable from this file's own public API.

**Resource leak on error path**
Reason: Both ETS tables and the code-server module registration are released in the `after` block, and `build/4` deletes the tables itself if `defmodule` reports an in-progress conflict.
Axes examined: exception during `eval_form`, exception in the spawned `elixir_erl_compiler` fun, `{error, module_in_definition}` from code server | Not examined: none
Basis: verified

**Silent corruption**
Reason: `cache_env/1`'s consecutive-dedup only reuses the immediately-previous key when the stored value is identical, otherwise allocates a fresh key — worst case a duplicate entry (harmless), never a wrong env returned by `get_cached_env/1`.
Axes examined: same env twice in a row (reuse), different env (new key), interleaved envs (new key, no false reuse) | Not examined: none
Basis: verified

**Infinite loop / hang**
Reason: The only unguarded `receive` is `make_module_available/1:623` waiting for `{Ref, ack}` from the compiler PID; that PID is the coordinating parallel-compiler process whose liveness is a compile-session invariant, and the same synchronous handshake is used throughout the compiler.
Axes examined: `elixir_compiler_info` undefined (skips receive), present (waits) | Not examined: compiler PID dying mid-handshake
Basis: asserted
Depends on: `elixir_compiler` — the process stored in `elixir_compiler_info` always sends `{Ref, ack}` or dies (which would propagate an exit, not hang the whole node); this is the established compiler coordination contract, not something this file can guarantee alone.

**Quadratic-where-linear**
Reason: Accumulators use `[X | Acc]` (`set_exports_md5`, `validate_inlines`); no `++` in a recursive loop and no `length/1` in a guard. `prune_stacktrace/2` is linear in stack depth.
Axes examined: `set_exports_md5` fold, `validate_inlines` recursion, `attributes` list comprehension | Not examined: none
Basis: verified

**Repeated work**
Reason: No pattern/regex compilation; `bag_lookup_element(DataBag, {accumulate, after_verify}, 2)` is looked up twice (once for tracing, once for the final verify loop) but each is a single ETS read, not a per-iteration recompile.
Axes examined: after_verify double lookup, config reads | Not examined: none
Basis: verified

**Dead / unreachable code**
Reason: All clauses are reachable; the `_ -> {"", true}` fallback in `beam_location`, the `false` validator clauses, and both stacktrace branches in the `undef` handler each correspond to a real runtime shape.
Axes examined: `elixir_compiler_dest` set vs unset, bootstrap vs non-bootstrap branches | Not examined: none
Basis: verified

**Documented behaviour mismatch**
Reason: There are no `-spec`s or `@doc` prose in this file to contradict; behaviour was checked against inline comments (table-layout comment at 400-424 matches the inserts) and the `format_error/1` strings match their raising sites.
Axes examined: every `format_error` tag vs its emit site, table-layout comment vs `build/4` inserts | Not examined: none
Basis: verified

**Race condition / shared mutable state**
Reason: The per-module ETS tables are process-owned within one compile job; the only cross-process state (`elixir_modules` registration) is serialized through `elixir_code_server:call({defmodule,...})`/`{undefmodule,...}`.
Axes examined: concurrent define of same module (code server returns `module_in_definition`), table ownership | Not examined: behaviour of the code-server gen_server under concurrent calls
Basis: asserted
Depends on: `elixir_code_server` — `{defmodule, Module, self(), Tuple}` atomically detects an in-progress duplicate and returns `{error, {Module,...}}`; this file relies on that call being the serialization point.

**Wrong scope / order of operations**
Reason: Side effects are correctly ordered — module name validated before `build/4`, `check_module_availability` before table creation, `make_readonly` before type inference, taint checked both before inference and immediately before `elixir_erl:compile`.
Axes examined: validation-before-build, taint-check placement (198 and 231), readonly before infer | Not examined: none
Basis: verified

**Encoding / unicode mishandling**
Reason: Module-name-to-path uses `elixir_utils:characters_to_list/1` on the dest and `atom_to_list/1` on the (atom) module; no byte-vs-codepoint confusion introduced here.
Axes examined: dest as binary, module charlist concatenation with ".beam" | Not examined: multi-byte module atoms (handled by `elixir_utils`)
Basis: asserted
Depends on: `elixir_utils:characters_to_list/1` — correctly converts chardata to a codepoint list for `filename:join`.

**Unclassified**
Reason: No file-specific composite issue surfaced; the taint/cache/counter ETS idioms are internally consistent and the compile teardown is exception-safe.
Axes examined: taint round-trip, cache-env dedup, compile error/after teardown | Not examined: none
Basis: verified

### Did I verify this?

- **Missing attribute-validation shapes (asserted):** kept asserted. Re-reading `validate_dialyzer_attribute`/`validate_definition` confirms they pattern-match `{Key,Funs}` and `{Name,Arity}` tuples, but the axis "an upstream inserts a malformed accumulated attribute" is by definition out of this file — it requires `Elixir.Module`/`elixir_def` misbehaviour, so it stays a named cross-file dependency, not an in-file finding.
- **`make_module_available/1` receive hang (asserted):** kept asserted. The untried axis (coordinator PID dying between send and ack) cannot be exercised from this file; a dead coordinator yields an exit signal to this linked compile process rather than a silent node-wide hang, and the handshake matches the rest of the compiler — verification belongs in `elixir_compiler`'s report.
- **Code-server serialization (asserted):** kept asserted. Whether `{defmodule,...}` is truly atomic under concurrency is a property of the `elixir_code_server` gen_server, not observable in this file; named as a `Depends on:` for the cross-file pass.
- **`elixir_utils` unicode conversion (asserted):** kept asserted for the same reason — the conversion correctness lives in `elixir_utils`.
- All classes marked `verified` above have empty `Not examined` lists and were traced against the concrete clauses cited; re-reading did not reveal an untried in-file axis for any of them.

