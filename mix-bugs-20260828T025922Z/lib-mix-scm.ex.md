# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/scm.ex
<!-- sha256: 7ea71213def6e14ca26590d28da4b9fd69f369c5ba002520215b1e7e51c1aedf -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/scm.ex
<!-- sha256: 7ea71213def6e14ca26590d28da4b9fd69f369c5ba002520215b1e7e51c1aedf -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`elixir/lib/mix/lib/mix/scm.ex` defines the `Mix.SCM` behaviour plus four trivial public helpers (`available/0`, `delete/1`, `prepend/1`, `append/1`) that wrap `Mix.State`. The executable logic is correct. The one defect is a copy-paste error in the `format_lock/1` callback documentation, now confirmed by reading the concrete `Mix.SCM.Git` implementation. The prior "out of scope" hedges are resolved: I read `Mix.State` (`state.ex`) and both `Mix.SCM.Git`/`Mix.SCM.Path`. The list-mutation atomicity hedge is upgraded to a low-confidence finding: `Mix.State.update/2` is a genuinely non-atomic read-modify-write.

## Bugfixes

#### `CSCAN-scm-format-lock-doc-copypaste`
- **Finding ID** — `CSCAN-scm-format-lock-doc-copypaste`
- **Severity** — Low
- **Confidence** — high
- **Location** — `scm.ex:36-44`, `@callback format_lock(opts)`
- **Class** — Documentation mismatch
- **Impact** — The published `@doc` for the `format_lock` callback is verbatim the text for `format` ("Returns a string representing the SCM. This is used when printing the dependency and not for inspection…"), which describes formatting the SCM *itself* rather than the *lock*. An SCM author reading the generated behaviour docs is told the wrong purpose for the callback; only the trailing "If nil is returned, it means no lock information is available." hints at the true intent. No runtime failure results, but the contract text is wrong.
- **Evidence** — verified against the neighbour implementation. `Mix.SCM.Git.format/1` (`scm/git.ex:15-21`) renders the *SCM descriptor* (`redact_uri(opts[:git]) - rev`), while `Mix.SCM.Git.format_lock/1` (`scm/git.ex:24-38`) renders *lock* content: it destructures `opts[:lock]` (`{:git, _, lock_rev, lock_opts}`), slices the resolved revision to 7 chars, and appends the ref/branch/tag. `Mix.SCM.Path.format_lock/1` (`scm/path.ex:20`) returns `nil` (no lock info). The two callbacks are therefore semantically distinct, and the shared first paragraph is a copy-paste of `format`'s text.
- **Suggested fix** — Replace the first paragraph of the `format_lock` doc with text describing that it returns a concise string representation of the *lock* (e.g. the resolved revision), keeping the "If nil is returned…" sentence.

#### `CSCAN-scm-state-update-nonatomic-rmw`
- **Finding ID** — `CSCAN-scm-state-update-nonatomic-rmw`
- **Severity** — Low
- **Confidence** — low
- **Location** — `scm.ex:142-158` (`delete/1`, `prepend/1`, `append/1`) via `Mix.State.update/2` (`state.ex:41-44`)
- **Class** — State-threading / concurrency (lost update)
- **Impact** — **Upgraded from hedged assumption in first pass** (prior report deferred atomicity to "outside this file's scope"). `Mix.State.update/2` is `:ets.insert(@name, {key, fun.(:ets.lookup_element(@name, key, 2))})` — a read (`:ets.lookup_element`) followed by a separate write (`:ets.insert`), with no serialization through the owning GenServer (the write bypasses `@name`'s process and hits the `:public` ETS table directly). If two processes concurrently call `prepend/1`/`append/1`/`delete/1` on `:scm`, one update can be computed against a stale list and overwrite the other, silently dropping a registered SCM. In the normal Mix lifecycle SCM registration happens single-threaded at startup, so this is not reachable through documented usage; hence low confidence. I did not execute a concurrent race to demonstrate a lost update, so the practical reachability remains unproven.
- **Suggested fix** — If concurrent SCM registration is ever intended, route list mutations through a serialized `GenServer.call` (or use `:ets.select_replace`/an atomic update) rather than a bare read-modify-write. Otherwise document that these helpers are startup-only and not concurrency-safe.

## Risks / caveats

- The four public helpers are one-line delegations to `Mix.State`. `delete`/`prepend`/`append` each `List.delete(&1, mod)` before re-inserting, so `prepend`/`append` correctly move an existing entry and avoid duplicates; their per-call logic is correct (the only cross-call concern is the non-atomicity above).
- All remaining entries in this file are `@callback` declarations with no implementation, so there is no behavioural code here to crash. Concrete implementations `Mix.SCM.Git` and `Mix.SCM.Path` were read this pass to confirm the `format_lock` doc finding; a full contract review of them belongs to their own files.

### Ruled out
- **`available/0` returning a wrong value if `Mix.State` were unstarted** — Confirmed by reading `state.ex:29-34` and `:71`: `get/1` reads `:ets.lookup(@name, key)`. If the `Mix.State` ETS table does not exist (Mix not started), `:ets.lookup` raises `ArgumentError` rather than returning `nil`; the `:scm` key is seeded at `init/1` (`state.ex:77`) so a started Mix always returns the `[Mix.SCM.Git, Mix.SCM.Path]` list. Reachable only by bypassing the documented `Mix.start()` lifecycle, which raises loudly rather than silently mis-returning. Axes examined: lifecycle-order (unstarted vs started), key-presence. | Not examined: none.
- **`delete/1`/`prepend/1`/`append/1` guard `when is_atom(mod)`** — passing a non-atom violates the documented `t :: module` contract and raises `FunctionClauseError`, the intended rejection of contract-violating input, not a bug. Axes examined: non-atom argument type. | Not examined: none.
- **Duplicate-entry buildup in `prepend`/`append`** — both call `List.delete(&1, mod)` before re-inserting (`scm.ex:150,157`), so no duplicates accumulate. Verified by inspection. Axes examined: repeated-registration of the same module. | Not examined: none.

### Did I verify this
- **`format_lock` doc finding** — Verified against neighbours: read `scm/git.ex:15-38` and `scm/path.ex:15-21`. `format` renders the SCM descriptor (URI), `format_lock` renders lock content (revision + ref/branch/tag) or `nil`. The shared doc paragraph is confirmed a copy of `format`'s text. Static text claim; no execution needed beyond reading the two implementations.
- **`Mix.State.update/2` non-atomicity** — Verified by reading `state.ex:41-44`: it is a two-call read-modify-write (`:ets.lookup_element` then `:ets.insert`) against a `:public` table (`state.ex:71`), not routed through the GenServer. The *structural* non-atomicity is confirmed; I did **not** run a concurrent race to observe an actual lost update, so the finding is held at low confidence on reachability grounds.

### Dropped in re-review
- None. All prior findings are preserved; the two prior "scope" hedges were resolved by reading the neighbour modules (`state.ex`, `scm/git.ex`, `scm/path.ex`) rather than dropped.
