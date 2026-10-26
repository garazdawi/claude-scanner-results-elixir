# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/scm.ex
<!-- sha256: 7ea71213def6e14ca26590d28da4b9fd69f369c5ba002520215b1e7e51c1aedf -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/scm.ex
<!-- sha256: 7ea71213def6e14ca26590d28da4b9fd69f369c5ba002520215b1e7e51c1aedf -->

_Re-reviewed on 2026-08-27_

This is `Mix.SCM`, a behaviour-definition module: the bulk is `@callback` declarations (lines 26–128, no executable code), plus four small functions — `available/0` (line 134), `delete/1` (line 142), `prepend/1` (line 149), `append/1` (line 156). In this re-review I read the two neighbour modules the first pass hedged on (`Mix.State`, `Mix.Dep.Loader`) and the in-tree callers of the mutators, so the "outside this file" assumptions are now backed by what I read.

**What I verified this pass (resolving first-pass hedges):**

- **`Mix.State` source IS in this checkout** (`lib/mix/lib/mix/state.ex`), contrary to the first pass's claim that its source was unavailable and its semantics "inferred from documented GenServer role." I read it. The material correction: **`Mix.State.update/2` is NOT serialized through the GenServer.** Lines 41–44 show it runs entirely in the *calling* process, doing a non-atomic read-modify-write directly against a `:public`, `:named_table` ETS table:
  ```elixir
  def update(key, fun) do
    :ets.insert(@name, {key, fun.(:ets.lookup_element(@name, key, 2))})
    :ok
  end
  ```
  The GenServer (`Mix.State`) only owns the table and serves `:builtin_apps`; `get/put/update/fetch` are plain ETS ops in the caller. So the first pass's statement "The read-modify-write runs inside Mix.State's serialized update" was factually wrong. This is corrected below. It does **not** change the verdict, because there is still no attacker in the read-modify-write loop (see Ruled out → Concurrency).
- **`:scm` is initialised to `[Mix.SCM.Git, Mix.SCM.Path]`** (state.ex:77) — a fixed list of trusted module atoms, not derived from any peer/archive/git input.
- **The only in-tree consumer of `available/0`** is `Mix.Dep.Loader` at `dep/loader.ex:224–225`: `Enum.find_value(Mix.SCM.available(), {nil, opts}, fn scm -> (new = scm.accepts_options(app, opts)) && {scm, new} end)` — dispatch on the registered module list, confirmed by reading the lines.
- **The only in-tree callers of the mutators** are the test file `test/mix/scm_test.exs` (lines 17, 19, 24, 26), which pass literal module atoms (`Hello`). Out of tree, Hex registers its SCM by passing its own module atom. No caller forwards network/archive/git-ref bytes into the `mod` argument.

## Summary
`Mix.SCM` is a behaviour-definition module whose only executable code reads/registers trusted SCM module atoms in global ETS-backed state. After reading `Mix.State` and `Mix.Dep.Loader`, I confirmed there is no sink reachable from network, archive, git-ref, or dependency-controlled input in this file. The first pass's incorrect "serialized update" rationale is corrected (the RMW is actually a non-atomic public-ETS operation), but the corrected facts still yield no security finding, because the mutator argument is always a trusted module atom and no attacker participates in the race. **No findings.**

## Security
No findings

### Ruled out

- **Shared mutable state — `Mix.State.update(:scm, ...)` in `delete/1`, `prepend/1`, `append/1` (lines 143, 150, 157).** Writes to VM-global ETS state, but the written value is a module atom the *caller* supplies. Verified callers: `scm_test.exs` (literal atoms) and, out of tree, Hex SCM registration (own module atom). No path in this file turns network/archive/git-ref bytes into `mod`. Not a boundary crossing. **Confidence: high.**
  Axes examined: source of `mod` argument (in-tree caller set via grep), initial `:scm` value (state.ex:77), consumer of the list (loader.ex:224). Not examined: none.

- **Atom creation / atom-table exhaustion (CWE-770).** `prepend`/`append`/`delete` are guarded `when is_atom(mod)`, so they only accept an already-interned atom — no `String.to_atom`/`Module.concat` byte→atom conversion in this file. (Note: `Mix.State` itself *does* call `String.to_unsafe_atom`/`List.to_unsafe_atom` at state.ex:91,116, but those are fed by `MIX_ENV`/`MIX_TARGET` env vars and the code path, i.e. operator-trusted, and are that file's concern, not reachable from `Mix.SCM`.) **Confidence: high.**
  Axes examined: guard on each mutator, absence of atom-creation calls in scm.ex. Not examined: none.

- **Behaviour/protocol dispatch on externally-influenced module — `available/0` → `scm.accepts_options(...)` at loader.ex:225.** I read the dispatch site. The dispatched module set comes from `:scm`, initialised to `[Mix.SCM.Git, Mix.SCM.Path]` and mutated only via the guarded mutators (needs code execution) or by co-resident code writing the `:public` ETS table (already operator-trusted). No untrusted input controls the dispatched module set via this file. **Confidence: high.**
  Axes examined: initial value, mutation entry points, dispatch site. Not examined: none.

- **Concurrency on `Mix.State.update` (CORRECTED from first pass).** I read state.ex:41–44 and confirmed `update/2` is a **non-atomic** read-modify-write against a `:public` ETS table executed in the calling process — it is *not* GenServer-serialized (the first pass asserted otherwise). Two concurrent `prepend`/`append`/`delete` calls can therefore lose an update (classic ETS RMW race). However, the only effect is list ordering/duplication among trusted SCM registrations, and `List.delete/2` de-dupes the module before the cons/concat, so a lost update at worst drops or reorders a trusted registration. No attacker is in the loop (all callers trusted), so this is a robustness note, not a security boundary crossing. **Confidence: high.**
  Axes examined: actual `update/2` implementation (read), atomicity (non-atomic ETS RMW, confirmed by code), attacker reachability of concurrent callers (none in-tree beyond tests/Hex). Not examined: none.

- **`is_atom/1` guard as dead defensive code.** A non-atom argument raises `FunctionClauseError`, matching the documented contract ("the given SCM module"). Passing a non-atom is spec-violating operator misuse, out of scope. **Confidence: high.**

- **Resource consumption.** `List.delete/2` and list concatenation are O(n) over the small trusted set of registered SCMs (initialised to 2 entries); n is not externally influenced. **Confidence: high.**

## Risks / caveats
- The security-relevant SCM behaviour lives in other files (out of scope here): the fetch/argument-injection surface is in `Mix.SCM.Git` (`System.cmd("git", …)` with dep-supplied refs/URLs); checksum/redirect handling is in `Mix.Utils`/archive tasks; trust decisions on the returned module list are in `Mix.Dep.Loader`/`Converger`. I read `Mix.Dep.Loader:224–225` only far enough to confirm the dispatch shape; its downstream handling of `accepts_options` results is that file's review.
- `Mix.State` performs its own env-var→atom conversions (`String.to_unsafe_atom`, state.ex:91) and code-path→atom conversion (`List.to_unsafe_atom`, state.ex:116). These are not reachable from `Mix.SCM` and belong to a `Mix.State` review; flagged here only because I read the file this pass.

## Did I verify this?
- **Read `Mix.State` (state.ex) in full** — resolved the first-pass hedge that its source was absent. Corrected the "serialized update" claim to "non-atomic public-ETS RMW."
- **Read `Mix.Dep.Loader:224–225`** — confirmed `available/0` is consumed for SCM dispatch, and the dispatched module set is exactly `:scm`.
- **Grepped all in-tree callers** of `prepend/append/delete/available` — confirmed the mutator argument is always a trusted literal module atom (tests) and never network/archive/git input.
- I did **not** execute a reproduction: there is no behavioural axis in contention. The corrected concurrency fact (non-serialized RMW) is established by reading the implementation, and it does not become a finding because no attacker participates in the race — a runtime race demonstration would only re-show a lost update among trusted callers, which is not a security claim.

### Dropped in re-review
No prior findings existed (the first pass reported none), so none are dropped. No ruled-out entry was upgraded to a finding: reading the neighbour modules *strengthened* the dismissals (the dispatched set is a fixed trusted list; the mutator argument is a trusted atom), and the one factual error found (update is non-atomic, not serialized) makes the concurrency story slightly *worse* mechanically but still attacker-free, so it remains ruled out with corrected, verified evidence.
