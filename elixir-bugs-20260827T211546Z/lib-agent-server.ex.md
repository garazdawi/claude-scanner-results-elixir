# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/agent/server.ex
<!-- sha256: 9c159588d727955def967abf1d610d23f19927bccb34219d0d891af813433205 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:33:44Z_

## Summary
`Agent.Server` is a faithful, correct implementation of the GenServer backing the public `Agent` API; I found no reachable bugs.

## Bugfixes

No findings

### Ruled out

- **`run/2` argument order (`extra ++ a`) for the MFA form** — For `Agent.get_and_update(agent, mod, fun, args)` the state must be prepended to the caller's args. `run({m,f,a}, extra)` does `apply(m, f, extra ++ a)` with `extra = [state]`, giving `mod.fun(state, args...)`, which matches the documented contract. Basis: `verified` — the MFA `get_and_update` test returned reply `[:seed]` (old state) and new state `[:x, :seed]`, exactly the expected `{state, [y|state]}` split.
- **`get_initial_call/1` non-total function** — The two clauses cover the only two shapes `init/1` ever receives: a 0-arity fun (from `Agent.start_link/2,3` with an anonymous function) and an `{mod, fun, args}` tuple (from `Agent.start_link(module, fun, args)`). No public `Agent` entry point delivers any other shape to `init`. Basis: `verified` — both a 0-arity-fun start and an MFA start succeeded and `$initial_call` was populated (`{:elixir_compiler_0, ...}` and implicitly for the MFA case).
- **`handle_call({:get_and_update, ...})` missing-clause / bad return** — A non-`{reply, state}` return falls to the `other` clause and cleanly `{:stop, {:bad_return_value, other}, state}`s, matching the documented Agent contract; there is no silent wrong return. Basis: `asserted` (both clauses present and exhaustive over any term; the `{reply, state}` path verified in the run above).
- **`get`/`update`/`cast` reply and threading** — `handle_call({:get,...})` replies with the fun result and preserves state; `{:update,...}` replies `:ok` and stores the new state; `handle_cast` stores the new state. Basis: `verified` — get→1, update→:ok then state 2, cast→state 100, all as expected.
- **Missing `handle_info`/catch-all** — Not a defect here: `Agent` documents no message protocol beyond the get/update/cast calls, and an unexpected message simply uses GenServer's default `handle_info` logging; no documented public path leaves a message stranded or crashes. Basis: `asserted` (no `handle_info` is part of the Agent contract; searched the whole file — no other message entry points exist).
- **State threading in `code_change/3`** — Applies the migration fun to the state and returns `{:ok, new_state}`, the correct `code_change` contract. Basis: `asserted` (single clause, correct shape; not driven, as hot code upgrade is not a scriptable public path).

## Risks / caveats
The file is 55 lines and appears to be an exact copy of the upstream `Agent.Server`; all five callbacks plus the two private helpers were reviewed and the get/update/get_and_update/cast/MFA paths exercised at runtime. I did not exercise the `code_change/3` hot-upgrade path (not reachable via a simple script), but its shape is correct by inspection.

