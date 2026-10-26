# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_config.erl
<!-- sha256: 4281426bd40a5e5ffe77a88c237b368593fca3dc47ac1ac1edce922a9bd032a7 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_config.erl
<!-- sha256: 4281426bd40a5e5ffe77a88c237b368593fca3dc47ac1ac1edce922a9bd032a7 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (single-file hedges resolved against callgraph neighbours)_

# Review: `elixir_config.erl`

1. **Summary** — The module is small and largely clean. The only substantive coded concern remains that `update/2` and `get_and_put/2` crash the shared config gen_server (rather than the caller) when handed a key that was never `put`. Re-review read the callgraph neighbours (`elixir.erl`, `elixir_compiler.erl`, `elixir_tokenizer.erl`) named in the three prior `Depends on` hedges; every cross-file dependency was **confirmed harmless**, so no dismissal was upgraded to a finding. Those entries stay `asserted` (not promoted to `verified`) because the axis rule requires executed differential evidence, which a neighbour read does not supply — but their `Depends on` claims are now cited to the exact lines that confirm them.

2. **Bugfixes**

#### `CSCAN-elixir_config-update-missing-key-crashes-server`
- **Finding ID** — `CSCAN-elixir_config-update-missing-key-crashes-server`
- **Severity** — Low
- **Confidence** — low
- **Location** — `elixir_config.erl:82-89` (`handle_call({update, ...})` and `handle_call({get_and_put, ...})`), via the internal call to `get/1` at `:24-26`.
- **Class** — Crash on stray / unexpected input
- **User-visible outcome** — `elixir_config:update(Key, Fun)` (or `get_and_put/2`) on a key that has never been `put` runs `Fun(get(Key))` / `get(Key)` inside the gen_server; `get/1` does `[{_, Value}] = ets:lookup(?MODULE, Key)`, which `badmatch`es on the empty list. Unlike a direct `get/1` call (which crashes only the caller), this crash happens **inside the singleton `elixir_config` gen_server**, so the server exits and is restarted by `elixir_sup`; the calling process receives an `exit` from `gen_server:call`, and any other request in-flight to the server is disrupted.
- **Reproduction** — With the config table started but no `foo` key inserted: `elixir_config:update(foo, fun(X) -> X end)`. The `get(foo)` call at `:83` evaluates `[{_,Value}] = ets:lookup(?MODULE, foo)` = `[]` → `{badmatch,[]}`, killing the server. A prior `put` of the key will not reproduce; the trigger is specifically the never-initialized key.
- **In-folder caller reachability (new in re-review)** — Grep over the folder shows **no in-folder caller of `update/2` or `get_and_put/2`**; the only `elixir_config:get/1` callers (`elixir_erl.erl:538,547`, `elixir_compiler.erl:11,19`, `elixir_expand.erl:496`, `elixir_module.erl:208,385,489`, `elixir_utils.erl:129`) all read keys that are inserted at startup in the `Config` list at `elixir.erl:82-102` (`debug_info`, `docs`, `parser_options`, `tracers`, `on_undefined_variable`, `erlc_options`, `ignore_module_conflict`, `module_definition`, `relative_paths`). So the crash is currently unreachable from in-folder code — hence confidence is **lowered from medium to low** versus the first pass. It remains a latent defect for any future/out-of-folder caller passing an uninitialised key.
- **Suggested fix** — If update/get_and_put are meant to tolerate absent keys, read with a default inside the handlers, e.g. use `get(Key, undefined)` (the crash-safe variant at `:28-34`) instead of `get(Key)` at `:83` and `:87`; otherwise document that the key must pre-exist.
- **Risk of fix** — Behaviour change: callers currently relying on `update`/`get_and_put` to raise on a missing key would instead see `Fun(undefined)` / `undefined`. Given `get/1`'s deliberate no-fallback design, confirm which contract is intended before changing.
- **Evidence** — read `elixir_config.erl:24-26,82-89`; the badmatch shape of `[{_,Value}] = ets:lookup/2` on `[]` is standard ETS behaviour; caller survey via grep over `elixir/lib/elixir/src`.

3. **Risks / caveats** — This file carries no `-spec`s, so the "spec-totality" checklist is vacuous and the update/get_and_put contract (must the key pre-exist?) is undocumented; the finding above is marginal precisely because the intended contract is unstated and no in-folder caller exercises the missing-key path. Table ownership / blast radius is now traced: the `?MODULE` ETS table is created by `new/1` from **`elixir:start/2`** (`elixir.erl:105`), owned by the application-start process — **not** by the gen_server (`init/1 :74-75` returns inert `[]`), so a gen_server crash does not by itself destroy the config table (see the "Resource leak" entry below).

### Ruled out

- **Operator / precedence typo**
  - **Reason** — The only boolean/counter expression is `warn/2`'s `ets:update_counter(?MODULE, EtsKey, {2,1,1,1}, {EtsKey,-1}) =:= 0`, which I traced call-by-call (first call → 0 → warn; subsequent → capped at 1 → no warn), giving the intended warn-once semantics.
  - Axes examined: first call (missing key, default inserted), second call, third+ call (threshold cap) | Not examined: none
  - **Basis** — verified

- **Missing clause / catch-all**
  - **Reason** — `handle_call/3` (`:77-89`) has no catch-all and would `function_clause` on an unexpected call tuple, but every call shape is produced only by this module's own `put/get_and_put/update/serial` wrappers, none of which are external message injectors; `handle_cast/2` (`:91-92`) intentionally `{stop,...}`s on any cast and no public function casts.
  - Axes examined: each documented call tuple (put/update/get_and_put/serial), unexpected cast | Not examined: hand-crafted `gen_server:call`/`cast` with a bogus tuple (not a documented API path)
  - **Basis** — verified

- **Wrong atom / wrong symbol**
  - **Reason** — persistent_term keys (`bootstrap`, `identifier_tokenizer` at `:18,:20`) and the default tokenizer atom `'Elixir.String.Tokenizer'` match the callers' expectations; no drifted record fields (module uses no records). **Cross-file dependency confirmed by reading the neighbours:** `identifier_tokenizer` is written by `elixir:start/2` at `elixir.erl:104` (`static(#{... identifier_tokenizer => Tokenizer})`) and read back under the identical atom key at `elixir_tokenizer.erl:128` (`elixir_config:identifier_tokenizer()`); `bootstrap` is written at `elixir.erl:104` (`false`) and `elixir_compiler.erl:169` (`true`) and read via `is_bootstrap/0` at `elixir_compiler.erl:40`, `elixir_erl.erl:162`, `elixir_module.erl:190,201,363,459`, `elixir_errors.erl:53,256`, `elixir_aliases.erl:11`, `elixir_lexical.erl:12` — all the same atom.
  - Axes examined: each maps:get key + default, ETS position literals in update_counter | Not examined: none via executed differential; the cross-module atom-key match is a static fact confirmed by neighbour reads, not by a value-push
  - **Basis** — asserted (kept in downstream passes per the "reading does not clear an axis" rule; cross-file `Depends on` is nonetheless confirmed below)
  - **Depends on** — `elixir_tokenizer.erl:128` / `elixir.erl:104`: they read `identifier_tokenizer`/`bootstrap` from persistent_term under exactly these atom keys — **confirmed by reading both.**

- **State / accumulator threading error**
  - **Reason** — gen_server `State` is the inert `[]` (`init/1 :75`) and is passed through every handler unchanged; there is no accumulator or St0/St1 rebind anywhere.
  - Axes examined: State value through serial/put/update/get_and_put clauses | Not examined: none
  - **Basis** — verified

- **Off-by-one / wrong arithmetic**
  - **Reason** — Only arithmetic is `to_arity/1` (`length/1` of an args list, `:66`) and the update_counter increment; both are correct.
  - Axes examined: empty args list (arity 0), non-empty list, bare arity integer | Not examined: none
  - **Basis** — verified

- **Contract violation**
  - **Reason** — Handlers return the documented shapes (`{reply, ok|Value|OldValue, State}`); `serial` returns `Fun()` directly as documented ("serial writes"). No `-spec` to contradict.
  - Axes examined: return shape of each handler | Not examined: none (no specs present)
  - **Basis** — verified

- **Crash on stray / unexpected input** — promoted to the finding above (`update`/`get_and_put` on missing key). No other stray-input crash: `get/2` wraps `ets:lookup` in `catch _:_ -> Default`, covering a missing table.

- **Resource leak on error path**
  - **Reason** — `new/1` (`:45-48`) creates the ETS table but ownership/cleanup is external. **Cross-file lifecycle confirmed by reading `elixir.erl:104-116`:** `new/1` is called once from `elixir:start/2` (`:105`); the OTP `start/2` return path deletes the table via `elixir_config:delete/1` on `elixir_sup:start_link` failure (`:111`) and `stop/2` deletes it on shutdown (`:116`). If `new/1` itself badmatches (its `ets:insert_new` at `:47` returning `false`), the freshly-created table is owned by the crashing `start/2` process and is auto-removed by ETS on that process's death — no leak. `insert_new` on a just-created empty table only returns `false` on duplicate keys within `Opts`, which the static `Config` list at `elixir.erl:82-102` does not contain.
  - Axes examined: ets:new + insert_new failure path (`:46-48`); caller cleanup path in `elixir.erl` (read) | Not examined: none via executed differential; the caller-cleanup fact is confirmed by neighbour read, not by a value-push
  - **Basis** — asserted (cross-file `Depends on` confirmed by reading; not promoted to verified absent executed evidence)
  - **Depends on** — `elixir.erl:105,111,116`: owns the config table lifecycle and tears it down on `start_link` failure / `stop` — **confirmed by reading; a `new/1` self-crash leaks nothing because ETS auto-deletes the owner's table.**

- **Silent corruption**
  - **Reason** — `static/1` merges into the existing persistent_term map (`maps:merge(persistent_term:get(?MODULE, #{}), Map)`) preserving prior keys; no truncation or mis-keyed writes.
  - Axes examined: merge with empty existing map, merge overwriting an existing key | Not examined: none
  - **Basis** — verified

- **Infinite loop / hang**
  - **Reason** — All `gen_server:call`s use `infinity`, but each handler replies synchronously with no `receive`/recursion; `serial`/`update` block only for the duration of `Fun()`. No catch-all returning `{noreply,...}`.
  - Axes examined: each handler's reply path | Not examined: a `Fun` that itself blocks forever (caller-supplied, contract)
  - **Basis** — verified

- **Quadratic-where-linear / Repeated work / Dead code**
  - **Reason** — No loops, no `++` accumulation, no pattern recompilation; every function is O(1) ETS/persistent_term access. No unreachable clauses (the oddly-indented `:79` clause is still a live, distinct clause).
  - Axes examined: each function body for iteration/compilation | Not examined: none
  - **Basis** — verified

- **Documented behaviour mismatch**
  - **Reason** — The only prose is `%% concurrent reads, serial writes` and the warn comment; reads (`get/1,2`) hit ETS directly (concurrent) and writes serialize through the gen_server, matching the comment. `warn/2`'s "once per caller" comment matches the traced counter semantics.
  - Axes examined: read-path concurrency, write-path serialization, warn-once | Not examined: none
  - **Basis** — verified

- **Race condition / shared mutable state**
  - **Reason** — `static/1` (`:15-16`) does an unserialized read-modify-write on persistent_term; concurrent callers could lose an update. **Cross-file confirmation by neighbour reads:** grep shows `static/1` is invoked only at `elixir.erl:104` (`elixir:start/2`, the single-threaded OTP application-start callback, before `elixir_sup:start_link`) and `elixir_compiler.erl:169` (`bootstrap/0`, a single-threaded setup routine that runs `ensure_all_started` then a straight-line sequence of `static`+`put` before any file compilation begins). No runtime/concurrent path calls `static/1`. Writes to the ETS store are correctly serialized via the gen_server.
  - Axes examined: concurrent static/1 (lost update), concurrent put via gen_server (serialized); all `static/1` call sites enumerated by grep and read | Not examined: none via executed differential; the "only single-threaded callers" fact is confirmed by neighbour reads, not by a value-push
  - **Basis** — asserted (cross-file `Depends on` confirmed by reading; not promoted to verified absent executed concurrency evidence)
  - **Depends on** — `elixir.erl:104` + `elixir_compiler.erl:167-169`: `static/1` is called only during single-threaded application start / bootstrap, never concurrently — **confirmed by reading both call sites (the only two in the folder).**

- **Wrong scope / order of operations**
  - **Reason** — `get_and_put` reads OldValue before inserting (`:87-88`) and returns the pre-write value, as its name implies; ordering is correct.
  - Axes examined: read-before-write ordering | Not examined: none
  - **Basis** — verified

- **Encoding / unicode mishandling**
  - **Reason** — Keys/values are opaque Erlang terms stored/fetched verbatim; no byte/codepoint interpretation occurs in this module.
  - Axes examined: term round-trip through ETS/persistent_term | Not examined: none
  - **Basis** — verified

### Did I verify this?

- **Wrong atom / wrong symbol** — cross-file `Depends on` now **verified by reading** `elixir.erl:104` (writer) and `elixir_tokenizer.erl:128` / the ten `is_bootstrap/0` call sites (readers); the atom keys `identifier_tokenizer` and `bootstrap` match on both sides. Entry stays `asserted` (not `verified`) only because the axis rule requires executed differential evidence, which a read is not; the dismissal is otherwise fully corroborated. No upgrade to a finding is warranted.
- **Resource leak on error path** — cross-file `Depends on` now **verified by reading** `elixir.erl:104-116`: the table lifecycle (create at `:105`, delete on sup-failure at `:111`, delete on stop at `:116`) is external and complete, and a `new/1` self-crash leaks nothing because ETS auto-deletes the crashing owner's table. Entry stays `asserted` per the axis rule; no leak found.
- **Race condition / shared mutable state** — cross-file `Depends on` now **verified by reading** the only two `static/1` call sites (`elixir.erl:104`, `elixir_compiler.erl:169`), both single-threaded startup/bootstrap paths. The lost-update window is real in isolation but unreachable concurrently. Entry stays `asserted` per the axis rule; no finding.
- All entries marked `verified` above had their `Not examined` list at `none` and were traced against the actual clause bodies in this file; no further axes remain for them.

### Dropped in re-review

- No prior findings dropped. The single finding (`CSCAN-elixir_config-update-missing-key-crashes-server`) is **retained** with confidence lowered from **medium to low**, because the re-review's caller survey showed no in-folder caller reaches the missing-key path (all `get/1` keys are startup-initialised in `elixir.erl:82-102`). It is kept, not dropped, as a latent defect for future/out-of-folder callers.

### Upgraded from hedged assumption in first pass

- None. All three first-pass hedged assumptions (`Wrong atom`, `Resource leak`, `Race condition`) were investigated by reading the named neighbours and each was **confirmed harmless**, so per "prefer downgrading to upgrading" none were promoted to findings. They remain `asserted` ruled-out entries (retained in downstream passes) with their `Depends on` lines now backed by cited neighbour reads rather than assumption.
