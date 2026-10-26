# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/on_exit_handler/supervisor.ex
<!-- sha256: 7b2896663bab0aa86e0420bd3243603bd7e94ffeb4c4152a2fb1a52c852294c6 -->

_Analysed on 2026-08-27T05:30:19Z_

## Summary

`ExUnit.OnExitHandler.Supervisor` is a 28-line internal (`@moduledoc false`) `:one_for_one` supervisor that ExUnit spins up per test process to hold `start_supervised` children, and it propagates the OTP `$callers` chain into the supervisor process so ownership/monitor semantics survive; every value reaching a sink here is either a hardcoded literal or an internally-derived list of pids, so there is no security-relevant exposure.

## Security

No findings

### Ruled out

- **`Supervisor.init(children, ...)` — child-spec / module dispatch (`supervisor.ex:15`).** `children` enters only via `start_link/1`, whose sole caller (`ExUnit.fetch_test_supervisor/0`, `ex_unit.ex:549`) passes a hardcoded `[]`. No external data reaches the child-spec list, so there is no attacker-controlled module/MFA dispatch. Even the hypothetical direct call `start_link(malicious_children)` is out of scope: the module is `@moduledoc false` (non-public entry point), and any such caller is the test author, who already has arbitrary code execution.
- **`:erlang.put(:"$callers", callers)` — shared mutable state (`supervisor.ex:26`).** The key is the literal atom `:"$callers"` (no dynamic atom creation), the value is `[self() | prior_$callers]` (pids only), and the write targets the freshly-spawned supervisor process's own dictionary. No externally-derived key or value; no cross-process/VM-global poisoning. This is the documented `$callers` propagation idiom.
- **`get_callers/1` — process-dictionary read (`supervisor.ex:18-23`).** Reads `:"$callers"` and prepends `owner = self()`. Both `case` clauses are reachable (`[_|_]` when a caller chain exists, `_` for the `undefined`/empty case), and a non-list value falls through safely to `[owner]` — robust, no crash, no dead defensive code.
- **`max_restarts: 1_000_000, max_seconds: 1` — resource consumption (`supervisor.ex:15`).** Deliberate near-infinite restart intensity (per the folder threat model) so a flapping on-exit/`start_supervised` child never trips the test supervisor. Config choice, not externally triggerable, and children start empty; not a security defect.
- **Check-then-act around `start_link` in `fetch_test_supervisor/0`.** A TOCTOU shape exists (`get_supervisor` → `start_link` → `put_supervisor`) but lives in another file and is benign: the ETS row is keyed by the caller's own pid and a single test process is sequential, so no racer can interleave meaningfully.

## Risks / caveats

- I traced `children` only through the single in-tree caller (`ex_unit.ex:549`, literal `[]`). A future caller that forwarded externally-influenced child specs into `start_link/1` would turn `Supervisor.init/2` into a module-dispatch sink; that is not the case today.
- No behavioural reproduction was warranted: there is no sink in this file that receives external/untrusted input, so there is no hostile-input claim to drive. All conclusions are by tracing (`asserted`), and rest on the assumption that OTP populates `$callers` with pids only — the standard, documented contract.

