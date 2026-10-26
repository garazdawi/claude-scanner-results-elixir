# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/agent.ex
<!-- sha256: 89540a0778f286a6f5d8be6e4fe675c7db709b2d24692959e28985be71e2a346 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:33:30Z_

All documented behaviors and doctests run exactly as specified. This is the canonical, thin `Agent`→`GenServer` delegation module.

## Summary
`agent.ex` is a thin, faithful wrapper over `GenServer` whose every public function and doctest behaves exactly as documented; I found no reachable bugs.

## Bugfixes

No findings

### Ruled out

- **`start_link/2` doc example invokes `Agent.start/1`, not `start_link` (line 274-276)** — The second doctest under `start_link/2` calls `Agent.start(fn -> raise "oops" end)`. This is intentional (the error-return shape only occurs unlinked; a linked failure would kill the doctest process) and the doctest passes as written. Basis: `verified` — ran it, output `{:error, {%RuntimeError{message: "oops"}, _stacktrace}}` matching the doctest.
- **`update/4`/`get/5`/etc. MFA arity — state prepended to args (lines 447, 356, 397, 492)** — Docs say "state is added as first argument to the given list of arguments." Verified `Agent.update(p2, Kernel, :+, [12])` yields state 54 from initial 42, confirming `Kernel.+(42, 12)`. Correct. Basis: `verified`.
- **Missing `is_function/1` guard on MFA-form functions (`get/5`, `get_and_update/5`, `update/5`, `cast/4`)** — By design; these take a module atom, not a function, so no guard applies. The anonymous-function heads (`get/3`, `update/3`, `cast/2`, etc.) each carry `is_function(fun, 1)` / `is_function(fun, 0)`. Not a defect. Basis: `asserted` (read all 4 MFA heads and all 6 fun heads in the file; guard presence/absence is correct for each arity).
- **`get_and_update/3` spec `(state -> {a, state})` vs implementation** — Server unpacks the 2-tuple; verified `{s, s+1}` returns the get-value and stores the new state (`gau: 42`, then `after gau: 43`). Matches spec and doctest. Basis: `verified`.
- **`__using__` `child_spec/1` generation + `defoverridable` (lines 212-233)** — Standard supervisor child-spec macro; `Macro.escape(opts)` correctly escapes user options into the quoted `Supervisor.child_spec/2` call, and `if not Module.has_attribute?(__MODULE__, :doc)` only injects a default `@doc` when the user hasn't set one, as the module doc at line 143-145 promises. No hygiene leak (uses `bind_quoted`). Basis: `asserted` (macro reviewed against documented contract; matches current upstream idiom).
- **`stop/3` default `timeout \\ :infinity` (line 515)** — Delegates to `GenServer.stop/3`; `:infinity` is the documented OTP default for synchronous stop. Not a hang bug — the caller explicitly requested a synchronous stop. Basis: `asserted`.
- **`child_spec/1` (line 204) hard-codes `id: Agent`** — This is the module-level `Agent.child_spec/1` (not the `use Agent`-generated one, which uses `__MODULE__`); using `Agent` as the id is correct here. Basis: `asserted`.

## Risks / caveats
- I reviewed only `agent.ex`. Its runtime correctness depends on `Agent.Server` (`lib/agent/server.ex`, out of scope); the get/update/cast tuple contracts (`{:get, fun}`, `{:get_and_update, fun}`, `{:cast, {m,f,a}}`, etc.) are honored by that module — confirmed indirectly since the behavioral tests passed, but the server file itself was not audited.
- The folder threat-model context (tokenizer/Unicode/Trojan-Source) is unrelated to this file; no identifier-security or normalization obligation applies to `Agent`.

