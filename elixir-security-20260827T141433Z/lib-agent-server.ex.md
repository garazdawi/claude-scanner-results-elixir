# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/agent/server.ex
<!-- sha256: 9c159588d727955def967abf1d610d23f19927bccb34219d0d891af813433205 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/agent/server.ex
<!-- sha256: 9c159588d727955def967abf1d610d23f19927bccb34219d0d891af813433205 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T15:00:00Z (first pass 2026-08-28T08:20:21Z)_

I re-read the entire file, read the sole hedged callgraph neighbour `lib/elixir/lib/agent.ex` in full, and re-ran the runtime probes against the built-from-source Elixir (`1.21.0-dev`, Erlang/OTP 29). The first-pass hedges keyed on "outside this file" all concerned the `Agent` public-API module; that module is now read and the trust-boundary claim is positively verified rather than assumed.

## Summary

`Agent.Server` is a thin, correct `GenServer` wrapper that runs developer-supplied functions/MFAs — the documented purpose of the `Agent` API. It creates no atoms, touches no files/network/shell, performs no byte→atom or deserialization conversion, and introduces no untrusted-data-to-sink path of its own. Reading its only caller (`Agent`) confirms the callables and MFA tuples are constructed from the developer's own arguments, not from attacker-shaped data. **No findings.**

## Security

`No findings`

The module's entire job is to `apply/2,3` a function or `{m, f, a}` supplied through the `Agent` public API (`Agent.get/update/get_and_update/cast` and their `module, fun, args` arities). Executing that code is the documented contract, not a vulnerability: the party supplying the callable is the developer, who already holds full code-execution in the VM. This file performs none of the dangerous conversions (no `String.to_atom`, `Module.concat`, `binary_to_term`, `Code.eval_*`, file/path/shell/network/crypto/unicode work).

- **Verified against the caller (`lib/agent.ex`), not assumed:** every arity that reaches `Agent.Server` either guards the callable with `is_function(fun, 0|1)` (`agent.ex:280,309,344,385,426,471`) or constructs the `{module, fun, args}` tuple itself from the developer's own positional arguments (`agent.ex:293,320,357,398,448,493`). The `Agent` module contains no `String.to_atom`/`Module.concat`/`:erlang.binary_to_term`/`Code.eval_*` — it never turns bytes into a module atom or a fun. So the "an external attacker would need a separate developer bug elsewhere" statement is now grounded in the read of the actual sole caller: no such conversion exists in the caller either; it would have to live in *application* code further up.

### Ruled out

- **Code execution via `apply/2,3` in `run/2` (`server.ex:53-54`)** — the applied `m/f/a` or `fun` originates from the `Agent` public API, which is the developer's own code; this is the intended behavior of `Agent`. *Basis: verified.* Read of `agent.ex` confirms the callable/tuple is developer-constructed with no untrusted-input conversion in the caller; and re-run `Agent.get(a, Probe, :combine, [:injected_arg])` returned `{0, :injected_arg}` — ordinary function application with caller-chosen `m,f`, no injection surface introduced here. Confidence: High.

- **Atom-table exhaustion (CWE-770)** — `get_initial_call/1` uses `Function.info/2` (returns pre-existing atoms) and destructures a caller-provided MFA (atoms already interned); no bytes→atom conversion exists in this file, and none in the caller (`agent.ex` has no `*.to_atom`/`Module.concat`). *Basis: verified.* Re-ran 5000 `Agent.update` calls: atom-count delta exactly `0` (`{20199, 20199, 0}`). Confidence: High.

- **`get_and_update` bad-return handling (`server.ex:19-24`)** — the `{:stop, {:bad_return_value, other}, state}` branch is a reachable, correct guard on the documented contract (the fun must return a 2-tuple); it stops only that one agent, triggered by the caller's own fun, not a cross-tenant attacker. *Basis: verified.* Re-ran a fun returning a non-tuple: target agent terminated with `{:bad_return_value, 0}` (`target_alive: false`), a sibling agent stayed up (`sibling_alive: true`), and the caller received a clean `:exit` carrying `{:bad_return_value, 0}`. Not dead code, not a DoS beyond the caller's own linked process. Confidence: High.

- **Process-dictionary write `Process.put(:"$initial_call", …)` (`server.ex:39`)** — writes to the agent's *own* process dictionary a value derived from the caller's fun/MFA; standard OTP process-identification convention (`Function.info` + MFA arity), not cross-process shared state and not externally poisonable. Confidence: High.

- **Message injection to `handle_call`/`handle_cast`** — reachable only by holding the agent pid and sending the internal `{:get, fun}` / `{:get_and_update, fun}` / `{:update, fun}` / `{:cast, fun}` shapes, i.e. via the documented `Agent` API (which requires in-VM code execution) or by hand-crafting raw messages to an internal process; the latter is out of scope per the review's internal-interface exclusion. Confidence: High.

- **Input-shape branching in `run/2` and `get_initial_call/1` (fun vs `{m,f,a}` tuple)** — *Basis: verified* against the caller. The tuple-vs-fun shape is fixed by *which* public `Agent` arity the developer calls: `agent.ex` constructs `{module, fun, args}` explicitly (`agent.ex:293,320,357,398,448,493`) and passes a bare fun for the anonymous arities. There is no public path that lets attacker-shaped data steer between the branches. (Note: `get_initial_call({mod, fun, args})` matches any 3-tuple, and a non-list `args` would crash in `length/1`; that is a developer misuse in the caller's process, not an attacker surface — `Agent.start(Kernel, :+, :not_a_list)` returns `{:error, …}` rather than corrupting state.) Confidence: High.

- **State disclosure via `handle_call({:get, fun}, …)` (`server.ex:15-17`)** — returns exactly what the caller's `fun.(state)` computes; the reply target is the calling process (`GenServer.call`). No leak beyond the API contract. Confidence: High.

## Risks / caveats

- `code_change/3` (`server.ex:34-36`) runs the fun supplied as the release-upgrade `Extra` term; that path is operator/release-controlled (a `:sys.change_code` / appup upgrade), which the threat model treats as already-privileged. Not an attacker surface. Confidence: High.
- Scope beyond the immediate caller: I verified `Agent.Server` and its sole caller `Agent` introduce no untrusted-input→callable conversion. If an *application* higher up deserializes funs from the network or builds module atoms from untrusted input and hands them to `Agent.*`, the defect lives in that application — not in this file and not in `agent.ex`. That is an application-code concern, not an unverified assumption about this file's neighbours.
- Verifications were re-run with `/opt/elixir/bin/elixir` (Elixir 1.21.0-dev / OTP 29, consistent with a build of this tree). Atom-count stability, single-agent stop behavior, sibling survival, and MFA application semantics are runtime-confirmed this pass.

## Did I verify this?

- **Read the hedged neighbour:** yes — `lib/elixir/lib/agent.ex` in full. It confirms guards (`is_function/2`) and developer-side tuple construction, and the absence of any atomization/deserialization in the caller. The first-pass "outside this file" hedges are resolved to *verified* by this read, not carried as assumptions.
- **Runtime evidence (re-executed this pass):** MFA injection is ordinary application (`{0, :injected_arg}`); 5000 updates → atom delta `0`; bad-return stops only the target (`target_alive: false`, `sibling_alive: true`) with a clean caller `:exit`.
- **Axes:** this report has no `Axes examined | Not examined` ruled-out entries to narrow; the dismissals rest on executed runtime tests plus the now-read caller, so none remain `asserted` on the strength of an unread neighbour.

## Dropped in re-review

None. Every first-pass finding-adjacent judgment (all "Ruled out") is preserved; the only change is upgrading the four "outside this file" hedges from assumptions to statements verified by reading `lib/agent.ex` and re-running the probes. No prior conclusion was reversed, and no new low-confidence finding was required because the hedged neighbour was readable and cleared the claim.
