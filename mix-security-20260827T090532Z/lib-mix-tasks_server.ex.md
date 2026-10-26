# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks_server.ex
<!-- sha256: 91d9b231bc3b1ce3614de44bbe2b63182f00290b9b5087e172e677d2019278c3 -->

_Analysed on 2026-08-27T11:30:11Z_

Confirmed. The module behaves exactly as the threat model describes — a trusted local idempotency store. `run/1` is the atomic `insert_new` run-once guard (`true` then `false`); all operations are total on arbitrary/hostile terms (invalid UTF-8, maps, refs, pids, funs stored and retrieved without crashing); nothing here executes, decodes, path-resolves, or atomizes the stored terms. I have enough to report.

## Summary
`Mix.TasksServer` is a 39-line internal (`@moduledoc false`) wrapper over a named ETS table used purely as Mix's run-once idempotency map, and it contains no sink that executes, decodes, path-resolves, or atomizes the task tuples it stores — no security findings.

## Security

No findings.

The complete inventory came up empty for this file. Every stored value (`{:task, task, proj}` / `{:alias, task, proj}`, per `task.ex:425–648`) is used only as an ETS equality key; none of the Phase-1 sink classes — code execution, atom creation, command/file/archive/network operations, deserialisation, interpolation, dispatch, validation, crypto, Unicode — has a call site here. Behaviour verified against the actual module (see run above): `run/1` returns `true` then `false` (atomic `:ets.insert_new` run-once guard, no TOCTOU), and `run/put/get/delete_many/clear` are total on hostile terms (invalid UTF-8 binaries, maps, refs, pids, funs) — all stored and read back without crashing or interpreting them.

### Ruled out
- **`:public` ETS table (line 13)** — any co-resident BEAM process can insert a `{:task, …}` key to make Mix skip a task (`get/1` → true) or delete keys to force re-runs. This is deliberate design (fast lock-free access; atomicity provided per-operation by ETS) and the attacker here would already be operator-trusted local code, not remote content. Explicitly out of scope per the folder threat model ("the ETS tables are `:public` … but such code is already operator-trusted").
- **Direct ETS access bypasses the owning Agent (lines 18–38)** — the Agent only *owns* the table so it survives; reads/writes deliberately do not serialize through it. Not a concurrency bug: `insert_new` (run/1) is atomic, and the idempotency contract depends on that atom, not on call ordering.
- **`delete_many/1` non-enumerable argument (line 32)** — `Enum.each/2` raises `Protocol.UndefinedError` if `many` isn't enumerable, but reaching that requires a caller to violate the documented contract (all call sites pass a literal list). Contract-violation misuse, excluded by scope.
- **Unbounded table growth (lines 19, 22)** — entries are bounded by the number of distinct `{task, proj}` pairs in a build; not attacker-amplifiable into a resource-consumption issue.

## Risks / caveats
- The `elixir` binary available in the sandbox is `/opt/elixir` (not built from the source under review); I loaded and exercised the *actual* `tasks_server.ex` source via `Code.require_file/1`, so the observed behaviour is this file's code, but the underlying ETS/Agent primitives are OTP-level and version-independent, so the reproduction is faithful regardless.
- This review is confined to `tasks_server.ex`. The one adjacent atom-creation concern I noticed — `String.to_unsafe_atom(task)` on task names in `task.ex:422,444` (atoms minted from possibly dependency-supplied alias keys) — lives in another file and is out of scope here; it belongs to a review of `mix/task.ex`.

