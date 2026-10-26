# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_config.erl
<!-- sha256: 4281426bd40a5e5ffe77a88c237b368593fca3dc47ac1ac1edce922a9bd032a7 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_config.erl
<!-- sha256: 4281426bd40a5e5ffe77a88c237b368593fca3dc47ac1ac1edce922a9bd032a7 -->

_Re-analysed on 2026-08-27T16:38:13Z_

Re-review of a compiler-internal config/state store (persistent_term flags + a public ETS key/value table serialized through a gen_server). Two prior ruled-out entries carried `Basis: asserted` because the deciding callers of `serial/1` and `update/2` live in the Elixir `.ex` stdlib, which is not present in this sandbox. Their sole `Not examined` axis is "exact stdlib call sites"; that axis cannot be *tried* here (the code is absent), and a grep of the in-folder tree confirms **no in-folder caller exists** to substitute for it. Because a dismissal cannot rest on an axis I could neither try nor confine in-folder, both are upgraded to low-confidence findings below. All other rulings — whose reasoning is contained in this file plus the in-folder call sites I read — remain as `verified` dismissals.

## Summary

`elixir_config.erl` is a compiler-internal config/state store whose ETS keys are compile-time-fixed atoms and whose values are operator/build-tool-controlled; no untrusted *source text* (the folder's primary attacker input) reaches any dangerous sink through the paths whose callers I can read. Two sinks — `serial/1` and `update/2` — invoke a caller-supplied `Fun()`/`Fun(get(Key))` *inside the global-config gen_server*, and their callers are outside this sandbox, so their fun-origin cannot be established here. They are recorded as low-confidence findings rather than dismissals.

## Security

### Config gen_server executes caller-supplied fun — `serial/1` origin unverifiable — `elixir_config.erl:55-56,77-78`
**Upgraded from hedged assumption in first pass.**
- `serial(Fun)` forwards an arbitrary zero-arity closure to the config gen_server, which runs it synchronously in the singleton's process: `handle_call({serial, Fun}, _, State) -> {reply, Fun(), State}`. A crashing, blocking, or looping `Fun` would take down or stall the global compiler-config singleton (`elixir_config` is a permanent `one_for_one` worker; its death aborts in-flight compilation state per the folder threat model). A `Fun` derived from untrusted source would additionally be arbitrary code execution in the config process.
- **Why this is a finding, not a dismissal:** there are **no in-folder callers** of `serial/1` (grep over `elixir/lib/elixir/src` returns nothing). The deciding callers are Elixir `.ex` stdlib modules absent from this sandbox, so the fun's origin — programmer-fixed closure vs. anything source-influenced — cannot be established here. The `Not examined` axis ("exact stdlib call sites") cannot be *tried* because that code is not present; reading in-folder neighbours cannot substitute for it.
- **Confidence: Low.** In-folder evidence (fixed keys everywhere else, gen_server serialization, compiler-internal naming) makes a benign fixed-closure contract the most likely reality, but I have not read a single caller to confirm it. Resolve in the cross-file pass by reading the `.ex` callers (`Code`/`Module`/`Kernel` and friends) and confirming every `:elixir_config.serial/1` argument is a programmer-defined closure never constructed from tokenized source.

### Config gen_server executes caller-supplied fun over a stored value — `update/2` origin unverifiable — `elixir_config.erl:42-43,82-85`
**Upgraded from hedged assumption in first pass.**
- `update(Key, Fun)` forwards `Key` + closure to the gen_server, which computes `Value = Fun(get(Key))`, then `ets:insert(?MODULE, {Key, Value})`, all inside the singleton. Same failure modes as `serial/1`: a crashing/looping `Fun` disrupts the global-config process; a source-derived `Fun` is code execution there. `get(Key)` inside also inherits the `get/1` badmatch (see caveat) if `Key` is absent.
- **Why this is a finding, not a dismissal:** identical to `serial/1` — **no in-folder callers** of `update/2`; the callers (e.g. `Code.put_compiler_option/2` et al.) are stdlib `.ex`, absent here. Neither the fun-origin nor the key-origin axis can be tried in this sandbox.
- **Confidence: Low.** `Key` is overwhelmingly likely a fixed atom and `Fun` a fixed closure given every in-folder usage pattern, but no caller was readable to confirm. Cross-file pass: confirm all `:elixir_config.update/2` call sites pass fixed atom keys and programmer-defined closures.

## Risks / caveats

- **Robustness, not security:** `get/1` (`elixir_config.erl:24-26`) hard-matches `[{_, Value}] = ets:lookup(...)` and crashes (`{badmatch,[]}`, reproduced below) if the key is absent. All *in-folder* call sites pass compile-time-fixed atom keys (`debug_info`, `docs`, `erlc_options`, `tracers`, `parser_options`, `on_undefined_variable`, `module_definition`, `ignore_module_conflict`, `relative_paths`, `infer_signatures`) inserted at bootstrap/start, so this is a compiler-internal invariant on the paths I can read. Note `update/2` calls `get(Key)` internally, so its (out-of-sandbox) key origin also feeds this.
- **Co-tenant fragility is out of scope per the folder threat model:** the ETS table is created `public` (`new/1`, line 46), so any BEAM process can write to it, and the gen_server takes itself down on an unknown message (`handle_call` function_clause; `handle_cast` → `{stop, {bad_cast, ...}}`, lines 77-92). Both require a hostile *co-tenant process* — explicitly not a boundary the compiler defends.
- The `identifier_tokenizer` value (read in `elixir_tokenizer.erl:128` for identifier-tokenization dispatch) is set once from `code:ensure_loaded('Elixir.String.Tokenizer')` in `elixir.erl:66-69` — a fixed module name, never influenced by source.

### Ruled out

**`get_and_put(Key, Value)` — `elixir_config.erl:39-40,86-89`**
- Reason: read-modify-write serialized through the gen_server; stores an operator/build-tool config value under a fixed key; no executable/atomizing sink. Unlike `serial`/`update`, this sink takes a *value*, not a fun — nothing is invoked.
- Axes examined: key origin, value origin, serialization (gen_server-mediated) | Not examined: none
- Basis: verified

**`put(Key, Value)` → `ets:insert` (shared mutable state) — `elixir_config.erl:36-37,79-81`**
- Reason: only in-folder writer is `elixir_compiler.erl:170-176` with fixed atom keys and constant values (bootstrap); all writes serialized via the gen_server, so no lost-update race. Takes a value, not a fun.
- Axes examined: key origin (fixed atoms), value origin (constants/operator config), write serialization | Not examined: none
- Basis: verified

**`static(Map)` → `persistent_term:put` (global state + non-atomic read-merge-write) — `elixir_config.erl:15-16`**
- Reason: two callers only (`elixir_compiler.erl:169`, `elixir.erl:104`), both at bootstrap/application-start in a single process; the merge value is `bootstrap`/`identifier_tokenizer`, both fixed; no concurrency and not source-reachable.
- Axes examined: caller set (2), concurrency (bootstrap single-process), value origin, persistent_term cost (bootstrap-only, not in a loop) | Not examined: none
- Basis: verified

**`new(Opts)` → `ets:new(public)` + `ets:insert_new` — `elixir_config.erl:45-48`**
- Reason: called once from `elixir.erl:105` with the static `Config` proplist (fixed atom keys); `public` table is only a co-tenant-write surface, out of scope.
- Axes examined: caller (single, app-start), Opts origin (static proplist), table access mode | Not examined: none
- Basis: verified

**`get/1` badmatch on missing key — `elixir_config.erl:24-26`**
- Reason: crash requires a key absent from the table; every *in-folder* call site passes a compile-time-fixed atom key inserted at bootstrap/start. (The out-of-sandbox `update/2` key origin is tracked as a low-confidence finding above, not here.)
- Axes examined: key origin at all in-folder call sites (`elixir_erl`, `elixir_module`, `elixir_compiler`, `elixir_utils`, `elixir_expand`), source-derivation | Not examined: none
- Basis: verified (call sites read via Grep; crash shape reproduced)

**`get/2` (`try…catch _:_ -> Default`) — `elixir_config.erl:28-34`**
- Reason: broad catch returns `Default` on any error including table-not-found; no sink, defensive by construction.
- Axes examined: error coverage (table missing, empty result), key origin | Not examined: none
- Basis: verified

**`warn/2` → `ets:update_counter` (resource growth / injection) — `elixir_config.erl:58-64`**
- Reason: key tuple is built from a real stack-trace MFA (`Mod`,`Fun` are existing atoms — no `list_to_atom`); counter is bounded per code-site (once-only, confirmed below); `update_counter` takes a key + update tuple, not a match-spec, so no injection.
- Axes examined: atom creation (none), match-spec injection (n/a), unbounded ETS growth (bounded by number of distinct warning sites), counter semantics | Not examined: none
- Basis: verified

**`delete/1` → `ets:delete` — `elixir_config.erl:50-51`**
- Reason: clause only matches the literal `?MODULE`; argument is the fixed table name from `elixir.erl:111,116`.
- Axes examined: argument origin, clause guard | Not examined: none
- Basis: verified

**gen_server `handle_call` unknown-message function_clause / `handle_cast` `{stop, bad_cast}` — `elixir_config.erl:77-92`**
- Reason: taking the singleton down on an unexpected message requires a co-tenant process to send a crafted message to an internal registered name; excluded by the threat model.
- Axes examined: message shapes handled (serial/put/update/get_and_put), sender trust boundary | Not examined: none
- Basis: verified

**Concurrency of writes vs concurrent `get/1` reads — `elixir_config.erl:22-48,77-89`**
- Reason: all mutations (`put`/`update`/`get_and_put`/`serial`) funnel through the single gen_server process, so read-modify-write is race-free; `get/1`/`get/2` are lock-free concurrent reads on a `{read_concurrency,true}` table, which is the intended design.
- Axes examined: write serialization, RMW atomicity, read/write interleave | Not examined: none
- Basis: verified

### Did I verify this?

- The `verified`-basis rulings whose reasoning is fully contained in this file plus the confirmed in-folder call sites (`get_and_put`, `put`, `static`, `new`, `get/1`, `get/2`, `warn/2`, `delete/1`, gen_server message handling, write/read concurrency) were read directly here; call-site key/value origins were confirmed via Grep against `elixir.erl`/`elixir_compiler.erl`/`elixir_module.erl`/`elixir_erl.erl`/`elixir_utils.erl`/`elixir_expand.erl`. The `warn/2` once-only counter and the `get/1` `{badmatch,[]}` shape were reproduced (`warn/2 returns: true false false`; `get(missing) => {error,{badmatch,[]}}`). Their `Not examined` lists read `none`.
- The two previously `asserted` rulings (`serial/1`, `update/2`) are **no longer dismissals**. Their only `Not examined` axis — "exact stdlib call sites" — is untried and *untriable in this sandbox*: the `.ex` callers are not present, and a grep over `elixir/lib/elixir/src` confirms there are **no in-folder callers** to push values through instead. Reading in-folder neighbours cannot clear that axis (it tells me what a guard does, not which inputs a caller supplies). Per the re-review rule that an unclearable `Not examined` axis cannot back a `verified` dismissal, and that upgrading a dismissal to a low-confidence finding is the safe direction, both are now **Low-confidence findings** for the cross-file pass to resolve by reading the stdlib callers.

### Dropped in re-review

None. Every prior ruling is preserved: the ten `verified` dismissals are unchanged, and the two `asserted` dismissals were not deleted but *upgraded* (the safe direction) into low-confidence findings.
