# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_lexical.erl
<!-- sha256: e99caa2b6106e891a69ebbd44f2336de2d0b117d6c4398cecf08182acf5638d6 -->
<!-- tree: 3389f8405 2026-08-25 -->

# Review: `elixir_lexical.erl`
<!-- sha256: e99caa2b6106e891a69ebbd44f2336de2d0b117d6c4398cecf08182acf5638d6 -->

_Re-reviewed on 2026-08-28_

- **Summary** — Small, stateless lexical-tracking dispatcher matching upstream Elixir. No bug is reachable through its documented public API (`run/3`, `with_file/3`, `trace/2`, `format_error/1`) using only in-file logic and the four in-folder neighbours (`elixir_aliases`, `elixir_config`, `elixir_env`, `elixir_errors`), all of which I read. The only residual uncertainties are three cross-module contract assumptions on `?tracker` (= `'Elixir.Kernel.LexicalTracker'`, elixir.hrl:11), a module that is **not present in this checkout and is not an in-folder callgraph neighbour**. Because I can neither read nor execute against it, those three cannot be ruled out; per the re-review rules I have upgraded them from asserted dismissals to low-confidence findings rather than leave them as asserted hedges.

## Bugfixes

Three low-confidence findings, each **upgraded from a hedged assumption in the first pass** because their basis was an un-inspectable cross-module contract on `'Elixir.Kernel.LexicalTracker'`. None is an observed defect; each is an unverifiable-contract risk. Note: the first pass's `Depends on:` lines named this module `elixir_lexical_tracker`; the actual macro target is `'Elixir.Kernel.LexicalTracker'` (elixir.hrl:11), which does not exist anywhere in this checkout.

### 1. Wrong-atom / wrong-symbol — API/atom contract with `?tracker` unverifiable (Confidence: low)

**Upgraded from hedged assumption in first pass.**

- **Class:** Wrong atom / wrong symbol.
- **Location:** lines 14, 28, 36, 40–41, 45, 51–52, 57, 60, 63, 66, 69, 72, 86, 89, 96, 102, 113; `mode/1` at 77–78.
- **Claim:** `trace/2`, `run/3`, and `with_file/3` call `?tracker` with the function names `start_link/0`, `stop/1`, `alias_dispatch/2`, `remote_dispatch/3`, `add_export/2`, `import_dispatch/4`, `import_quoted/4`, `add_compile_env/4`, `set_file/2`, `reset_file/1`, `collect_unused_{imports,requires,aliases}/1`, and pass the mode atoms `compile`/`runtime` (from `mode/1`). Within this file these atoms and arities are internally consistent, and the env-key side (`from_macro` in `Meta`, `function`/`context` in `E`) is **verified** against `elixir_env:new()` (function/context default to `nil`). What is **not** verifiable is that `'Elixir.Kernel.LexicalTracker'` actually exports these names/arities and accepts `compile`/`runtime`. A mismatch would raise `undef`/`function_clause` at compile time.
- **Failure scenario:** If the tracker's public API renamed e.g. `add_export/2` or expected a mode atom other than `compile`/`runtime`, every affected compile would crash with `undef`/`function_clause`. Realistically such a break would fail the test suite, but I cannot confirm the contract from this checkout.
- **Why not ruled out:** `'Elixir.Kernel.LexicalTracker'` is absent here and is not a listed in-folder neighbour; I could not read it or run a call against it. Reading cannot clear this, and no execution is possible.
- **Cross-file confirmation needed:** the cross-file pass must confirm `'Elixir.Kernel.LexicalTracker'` exports exactly these names/arities and accepts the `compile`/`runtime` mode atoms.

### 2. Dead / unreachable code — `format_error/1` `unused_require` branches depend on tracker output shapes (Confidence: low)

**Upgraded from hedged assumption in first pass.**

- **Class:** Dead / unreachable code.
- **Location:** `format_error/1` lines 122–128 (the three-way `if`), fed by `warn_unused_requires/2` line 100–103 comprehension over `?tracker:collect_unused_requires/1`.
- **Claim:** The three branches (`Alias == false`, `AliasUsed`, else) are each reachable only if `collect_unused_requires/1` can emit all three `{Module, Meta, Alias, AliasUsed}` shapes — `Alias=false`; `Alias=module, AliasUsed=true`; `Alias=module, AliasUsed=false`. Within this file the `if` is a well-formed 3-way partition (this part is verified). Whether the tracker actually produces all three shapes is unverifiable here.
- **Failure scenario:** If the tracker never emits, say, the `AliasUsed=true` shape, the middle message ("convert it to an alias instead") is dead code — no crash, purely a coverage/maintenance concern.
- **Why not ruled out:** producing module absent; cannot inspect `collect_unused_requires/1` or run it.
- **Cross-file confirmation needed:** confirm `collect_unused_requires/1` can emit each of the three shapes.

### 3. Race / ordering of `set_file`/`reset_file` in `with_file/3` (Confidence: low)

**Upgraded from hedged assumption in first pass.**

- **Class:** Race condition / shared mutable state.
- **Location:** `with_file/3` lines 84–90 (`?tracker:set_file` / `?tracker:reset_file`).
- **Claim:** Correctness of the set-then-reset pair around the callback depends on `set_file`/`reset_file` being serialized per-tracker gen_server calls, and on each compile process owning its own tracker Pid, so concurrent `with_file` invocations on the *same* tracker cannot interleave file state. In-file, the `try/after` structure is correct (reset always runs). The concurrency safety is a property of the tracker module.
- **Failure scenario:** If two compile paths shared one tracker Pid and `set_file`/`reset_file` were not serialized/scoped, one path's `reset_file` could clobber the other's `set_file`, attributing a warning to the wrong file. No crash; wrong diagnostic location.
- **Why not ruled out:** tracker module absent; gen_server serialization and one-tracker-per-process ownership cannot be confirmed from this checkout.
- **Cross-file confirmation needed:** confirm `'Elixir.Kernel.LexicalTracker'` is a gen_server with per-Pid serialized `set_file`/`reset_file`, and that each compile process gets its own tracker (started at run/3 line 14).

## Risks / caveats

- The module has **no `-spec`s and no `@doc`**; the contract is inferred from callers (`elixir_compiler`, `elixir_def`, `elixir_module`) and `elixir.hrl`. Reachability depends on callers passing env maps containing `tracers`, `function`, `context`, `lexical_tracker` — an `elixir_env:new()` invariant now **verified** (elixir_env.erl:20–30). A future hand-built partial env would `function_clause`/`badmatch` at `run/3`'s head or the `alias_reference`/`mode` clauses, but that is caller contract violation, out of scope.
- **`run/3` bootstrap-return asymmetry** (lines 24 vs 33) is latent and currently masked (no consumer reads the return during bootstrap). Worth a one-line fix but bites nobody today — see Ruled out → Contract violation.

### Ruled out

**Operator / precedence typo**
- Reason: The only compound boolean/guard logic is `format_error/1`'s `if` (`Alias == false ; AliasUsed ; true`), a well-formed 3-way partition; `mode/1`/`alias_reference` are single-pattern clauses. No `,`/`;` guard mixing, no `andalso`/`orelse`.
- Axes examined: Alias=`false`, Alias=module + AliasUsed=`true`, Alias=module + AliasUsed=`false` | Not examined: none
- Basis: verified

**Missing clause / catch-all**
- Reason: `trace/2` has a `trace(_, _) -> ok` catch-all (line 74); `format_error/1` covers every tuple `warn_unused_*` constructs; `unused_imports_for_module/2` has a `#{}` fall-through. Additionally confirmed via `elixir_env:trace/2` (elixir_env.erl:33–35), which does `ok = Tracer:trace(Event, E)` — it forwards **arbitrary** event tuples and asserts an `ok` return, so the catch-all is both necessary and sufficient, and every `trace/2` clause does return `ok`.
- Axes examined: unknown trace event, unused_import as MFA tuple vs bare module, whole-module-unused vs per-function import map, non-`ok` return would badmatch in elixir_env:trace | Not examined: none
- Basis: verified

**State / accumulator threading error**
- Reason: `run/3` threads `LexEnv` (not the pre-mutation `E`) through execution, all three `warn_unused_*`, and the after-callback; `with_file/3` threads `E#{file := File}` into the callback and resets the tracker file in `after`. No stale-variable reuse.
- Axes examined: lexical_tracker=nil vs Pid, execution callback success vs raising (after-block cleanup) | Not examined: none
- Basis: verified

**Off-by-one / wrong arithmetic**
- Reason: No arithmetic, indexing, or length computation in the file.
- Axes examined: n/a | Not examined: none
- Basis: verified

**Contract violation (return-shape)**
- Reason: `run/3`'s bootstrap branch (lines 32–33) returns `AfterExecutionCallback(E)`; the non-bootstrap branch (line 24) returns `Res`. This asymmetry is real but harmless: `elixir_compiler.erl` discards the return (uses `get(elixir_module_binaries)`), and `elixir_module.erl` propagates it as `compile/6`'s result whose after-callback is `fun(_) -> ok end`, consumed only at runtime, not during bootstrap. `with_file/3` returns the callback result consistently in both clauses. Matches upstream.
- Axes examined: bootstrap=true (returns after-cb = `ok`) vs false (returns exec result); consumers in elixir_compiler (discards) and elixir_module (propagates but not read during bootstrap) | Not examined: none
- Basis: verified

**Crash on stray / unexpected input**
- Reason: Public entry points take env maps produced by `elixir_env` — confirmed to always carry `tracers`/`function`/`context`/`lexical_tracker` via `elixir_env:new()` (elixir_env.erl:20–30). `trace/2` cannot crash on unknown events (catch-all). No untrusted external byte/text input reaches this module (downstream of tokenizer/parser).
- Axes examined: unknown trace tuple, env with lexical_tracker nil vs Pid, env-key presence (verified against elixir_env:new) | Not examined: none (the previously-untried "env missing keys" axis is verified impossible for env-produced maps; a hand-built partial env is caller contract violation, out of scope)
- Basis: verified

**Resource leak on error path**
- Reason: `run/3` starts the tracker with `start_link` and always runs `unlink(Pid)` + `?tracker:stop(Pid)` in the `after` block; `with_file/3` resets the file in `after`. An exception in execution/after callbacks still tears down.
- Axes examined: execution callback raises, after-callback raises, warn_unused raises | Not examined: `?tracker:stop/1` itself raising (cross-module, module absent) — but a raise there would not leak (link already broken via unlink; the after already ran trace(stop)); the tracker is `start_link`ed so it also dies with the process on abnormal exit.
- Basis: verified

**Silent corruption**
- Reason: No data transformation that could return plausible-but-wrong values; `unused_imports_for_module/2` faithfully distinguishes whole-module (`#{Module := Meta}`) vs per-function (`#{{Fun,Arity} := Meta}`) unused imports.
- Axes examined: whole-module unused vs per-function import map | Not examined: none
- Basis: verified

**Infinite loop / hang**
- Reason: No recursion, no `receive`, no unbounded loops; all list comprehensions iterate finite tracker output.
- Axes examined: n/a | Not examined: none
- Basis: verified

**Quadratic-where-linear / Repeated work**
- Reason: The three `warn_unused_*` comprehensions each iterate the tracker's collected lists once; no `++`-in-loop, no pattern recompilation, no per-iteration `length/1`.
- Axes examined: large unused-import/alias/require sets | Not examined: none
- Basis: verified

**Documented behaviour mismatch**
- Reason: The `alias_reference` clause comment ("references inside patterns and guards in functions are not compile time dependencies") matches the code: `function=nil`→compile dep, `context=nil`→runtime dep, else→`ok` (no dep). No prose contradicts the implementation.
- Axes examined: function=nil, function set + context=nil, function set + context=match/guard | Not examined: none
- Basis: verified

**Wrong scope / order of operations**
- Reason: In `run/3` the tracker is started and `elixir_env:trace(start, …)` fired before the execution callback, stop/unlink in `after`; warnings emitted before the after-callback, all in intended order.
- Axes examined: start-before-exec, stop-in-after, warn-before-after-callback | Not examined: none
- Basis: verified

**Encoding / unicode mishandling**
- Reason: Module names/atoms are passed to `elixir_aliases:inspect/1` (confirmed accepts atoms, elixir_aliases.erl:10) and `io_lib:format` with `~ts` (unicode-aware). No raw source bytes handled.
- Axes examined: `~ts` vs `~s` in format_error, unicode module atom | Not examined: none
- Basis: verified

- **Race condition / shared mutable state** — **promoted to Bugfix #3 (Confidence: low)**; no longer an asserted dismissal. The tracker module (`'Elixir.Kernel.LexicalTracker'`) is absent from this checkout and not an in-folder neighbour, so its per-Pid serialization of `set_file`/`reset_file` cannot be confirmed. This module itself holds no ETS/process-dictionary state.

- **Wrong atom / wrong symbol** — **promoted to Bugfix #1 (Confidence: low)**; env/meta-key side verified against `elixir_env:new()`, but the `?tracker` API-name/mode-atom side is unverifiable (module absent).

- **Dead / unreachable code** — **promoted to Bugfix #2 (Confidence: low)**; the `format_error` `if` is a verified 3-way partition, but reachability of its branches depends on `collect_unused_requires/1` output shapes, unverifiable (module absent). (The trace-clause reachability sub-claim remains sound: `elixir_env:trace/2` forwards arbitrary events.)

### Did I verify this?

- **Operator/precedence, Missing clause, State threading, Off-by-one, Contract violation, Crash-on-stray, Resource leak, Silent corruption, Infinite loop, Quadratic, Documented behaviour, Wrong scope, Encoding** — `verified`. I read every clause (lines 11–128) and read all four listed callgraph neighbours: `elixir_env.erl` (confirmed `new()` env keys and the `ok = Tracer:trace/2` return contract), `elixir_errors.erl` (`file_warn/4` spec at line 251), `elixir_aliases.erl` (`inspect/1` atom guard at line 10), `elixir_config.erl` (`is_bootstrap/0` at line 17). The `run/3` return asymmetry was cross-checked against both consumers.
- **The three previously-asserted entries (Wrong atom, Dead code, Race condition)** — I could not verify them: `?tracker` = `'Elixir.Kernel.LexicalTracker'` is absent from this checkout and is not one of the four in-folder callgraph neighbours, so it cannot be read, and no differential/execution check is possible against it. Following the re-review rule ("prefer downgrading; upgrade the unverified assumption to a low-confidence finding"), I moved each from an asserted dismissal to a low-confidence finding rather than leave a hedge. Each names the exact cross-file confirmation the next pass must perform.

### Dropped in re-review

No prior findings were dropped. (The first pass had zero Bugfixes; this pass adds three low-confidence findings by upgrading the three asserted ruled-out entries the first-pass reviewer flagged, and strengthens several dismissals from `asserted`/partial to `verified` using the four neighbour modules.)
