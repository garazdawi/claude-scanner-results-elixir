# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/remote_converger.ex
<!-- sha256: 91cf5f5327ddeccae7e0edec78b18109be36cfd7b0ed4a785ab4e544e5123632 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/remote_converger.ex
<!-- sha256: 91cf5f5327ddeccae7e0edec78b18109be36cfd7b0ed4a785ab4e544e5123632 -->

_Re-analysed on 2026-08-27_

I re-read the full file and, to discharge the two hedges from the first pass ("out of scope for this file," "single-file review"), I read both callgraph neighbours that the hedges deferred to:
- `lib/mix/state.ex` — the storage backend for `get/0`/`register/1`.
- `lib/mix/dep/converger.ex` — the sole dispatch site for the stored module.

Both hedges were pointers to other files rather than untried input axes, so reading those files is the correct discharge here (there is no "Axes examined / Not examined" ruled-out entry in this file that would require me to execute an input to narrow it — the two functions are a fixed-key ETS store and read with no behavioural branch that could differ on hostile input).

## Summary
`Mix.RemoteConverger` is a 51-line behaviour + global-state shim. Its only two executable functions store (`register/1`) and read (`get/0`) a single fixed-key atom in `Mix.State`. Having now read both neighbours, I confirm: no untrusted data flows into `register/1` in this file, and the security-relevant dispatch on the stored module lives entirely in `dep/converger.ex`, driven by a value that only Hex (operator-trusted, compile-time constant) ever writes. No security findings.

## Security

No findings.

### Ruled out

- **Behaviour/module-dispatch RCE via a hostile registered module** — **Confidence: high (verified).** First pass deferred this as "the dispatch site is out of scope." I read `dep/converger.ex`: the stored module is dispatched at lines 75 (`remote.post_converge()`), 96 & 108 & 122 (`remote.remote?/1`), 118 (`remote.converge/2`), and 143 (`remote.deps/2`) — all via `Mix.RemoteConverger.get()`. Crucially, `converger.ex` never *calls* `register/1`; it only reads. The only writer of `:remote_converger` is Hex at startup, passing a compile-time module constant. So no attacker-influenced byte reaches `register/1` within this file, and reaching a hostile registered module requires the attacker to already run Elixir in the build — the "operator/dependency controls the build = RCE by design" case that is explicitly out of scope. Verified, not a finding.

- **Atom-table exhaustion (`register/1`)** — **Confidence: high (verified).** The clause is guarded `when is_atom(mod)` and performs no `String.to_atom`/`binary_to_atom`/`Module.concat`; the atom must already exist before the call. `Mix.State.put/2` (state.ex:36-39) is a plain `:ets.insert` of the `{key, value}` tuple — it creates no atoms either. No external-bytes-to-atom path. Not a finding.

- **Shared-mutable-state poisoning (`Mix.State.put(:remote_converger, mod)`)** — **Confidence: high (verified).** First pass deferred the storage semantics to `Mix.State` as out of scope. I read state.ex: the backing table is created `:public` (state.ex:71), so any co-resident BEAM code *can* overwrite the `:remote_converger` entry, and a poisoned entry would then be dispatched by `converger.ex`. However, the key here is a fixed literal and the value is a guarded atom supplied by trusted Hex; the value is not derived from network/disk/archive input in this file. The `:public`-ETS co-tenant write requires already-operator-trusted code running in the same VM — the excluded attacker model (mirrors the `[[partition-supervisor-review]]` / `Mix.State` `:public`-table caveat). Not a finding in this file.

- **`is_atom/1` guard as a dead/weak defensive check** — **Confidence: high (verified).** The guard is reachable (Hex passes a module atom; the test suite passes `nil`, which is an atom) and merely raises `FunctionClauseError` on non-atoms. It asserts arity/type, not that the module is safe to dispatch, so it is not a security predicate and its strength is not a security gap. Note the guard admits any atom, including a non-loaded module name — but a bad atom would surface only as an `UndefinedFunctionError` at the `converger.ex` dispatch site, from an operator-supplied value, not from untrusted data. Not a finding.

- **TOCTOU / init-once race between `get` and `register`** — **Confidence: high (verified).** `register/1` is a one-time startup write of a single fixed key; `Mix.State.put`/`get` are single `:ets.insert`/`:ets.lookup` calls (state.ex:29-39) with no check-then-act sequence and no trust boundary crossed. The `converger.ex` reader re-calls `get()` on every iteration (lines 96, 106) precisely because the converger may be lazily loaded — that is intended lazy-availability handling, not an exploitable race. Not a finding.

## Risks / caveats
- The genuinely security-relevant surface associated with the remote converger — the dynamic dispatch to the stored module and all handling of untrusted Hex registry data / `mix.lock` / dependency metadata — lives in `lib/mix/dep/converger.ex` (dispatch) and in the Hex package (untrusted data). I read `converger.ex` for the dispatch mechanics and confirmed it consumes the stored module but sources no untrusted data *into* this file's two functions; a converger-related vulnerability would surface in those consumers, not in this registration/lookup shim.
- No reproduction was run: neither function has a behavioural branch whose output could differ on hostile input (a guarded `:ets.insert` of a fixed-key tuple, and a fixed-key `:ets.lookup`). There is no behavioural claim to settle by execution, and no ruled-out entry here carries an untried input axis.

## Did I verify this?
- **`get/0` / `register/1` bodies and the `is_atom` guard** — yes, read directly (lines 40-49).
- **Storage semantics (`Mix.State.put`/`get`, `:public` ETS, no atom creation)** — yes, read `lib/mix/state.ex:22-39, 71`. Upgraded the first-pass "out of scope for `Mix.State`" hedge to a read-verified dismissal.
- **Dispatch site and the direction of data flow (reader vs. writer)** — yes, read `lib/mix/dep/converger.ex:75, 96, 106-108, 118, 122, 143`; confirmed it only calls `get()`, never `register/1`. Upgraded the first-pass "dispatch site is out of scope" hedge to a read-verified dismissal.

## Dropped in re-review
None. Every prior ruled-out entry is preserved; two of them (module-dispatch RCE, shared-state poisoning) had their first-pass out-of-scope hedges discharged by reading the named neighbour files and are now marked verified rather than asserted. No prior finding is dropped, and no dismissal was upgraded to a finding — the neighbour reads confirmed the shim carries no untrusted-data flow.
