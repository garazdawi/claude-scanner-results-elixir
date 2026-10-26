# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/supervisor.ex
<!-- sha256: 67fe4de593b77ddacd48b17932afd63a2a43ab728972b0014c734728994a382d -->

_Analysed on 2026-08-27T08:32:54Z_

I've now confirmed the full trust boundary. Let me record the key facts:

- `backend` reaches `Logger.Backends.Supervisor.add/1` and `remove/1` from exactly two places: (1) `Logger.add_backend/2` (documented public API) → `Logger.Backends.Internal.add/2` → here, where `backend` is a **module name / `{module, id}` tuple chosen by the developer's application code**; and (2) `Application.get_env(:logger, :backends, [])` at init, i.e. **operator config**. In neither case is `backend` derived from network/disk/dependency bytes that this code parses.

I have enough to judge every sink. No reproduction is needed because there is no untrusted-data-to-sink path to demonstrate; the one borderline item (the `add/1` recursion) is a concurrency/robustness question whose inputs are operator-controlled, so it falls outside a *security* review regardless of outcome.

---

# Security Review — `logger/backends/supervisor.ex`

## Summary
This 68-line internal (`@moduledoc false`) OTP supervisor manages the deprecated `:gen_event`-based Logger backends, and every value it acts on (`backend` module names, the `backends` list) is developer/operator-supplied configuration rather than attacker-forwarded data, so there are no security findings.

## Security
No findings.

Sink inventory and disposition (Phase 1 → Phase 2):

- **Protocol / behaviour dispatch** — `add/1` builds a child spec whose `start` is `{Logger.Backends.Watcher, :start_link, [{backend, backend}]}`, ultimately installing `backend` as a `:gen_event` handler module. `backend` enters only via `Logger.add_backend/2` (developer app code) or `config :logger, :backends` (operator config). Not externally influenced. No finding.
- **Atom creation** — `backend` is used as the child `id` and handler module, but it arrives already an atom/tuple from config; there is no `String.to_atom`/`Module.concat` on bytes. No finding.
- **Template / interpolation** — the `raise "... #{inspect(backend)} ..." <> Exception.format_exit(exit)` in `start_link/1` interpolates `backend` (via safe `inspect`) and a formatted exit reason into an exception message at trusted app-startup. No downstream interpreter (no shell/query/regex/log-format sink), and the value is operator config. No finding.
- **Concurrency (check-then-act)** — `add/1`'s `{:error, :already_present} → delete_child → add/1` recursion and `remove/1`'s terminate-then-delete are TOCTOU-shaped, but they are reachable only by concurrent operator/developer calls with operator-chosen `backend` values and operator-controlled timing. Not attacker-reachable; robustness, not security. See Ruled out.
- **Resource consumption** — `init/1` sets `max_restarts: 30, max_seconds: 3`. The value is a normal restart-intensity choice; no crash-inducing logic lives in this file. See Ruled out.

No code-execution, compile-time-execution, command-execution, file/path, archive, deserialisation, network, crypto, Unicode, round-trip, or shared-mutable-state-from-external-input sinks are present.

## Risks / caveats
- I traced the trust boundary one hop out (`logger.ex`, `backends/internal.ex`) and confirmed `backend`/`backends` originate from `Logger.add_backend/2` arguments and `Application.get_env(:logger, :backends)`. If some other undocumented caller were to route parsed untrusted bytes into `backend`, the behaviour-dispatch and child-`id` sinks would deserve re-examination — but no such caller exists in this application, and `Logger.add_backend/2`'s `@spec` types `backend` as a module/backend, so passing parsed foreign data would be a contract violation (out of scope).
- I did not run a reproduction: the conclusion is the *absence* of an untrusted-input path, which is established by the call-graph above rather than by observing a behaviour. The two "Ruled out" items below are the only behavioural questions, and both are out of scope for a security review because their inputs are operator-controlled.

### Ruled out
- **`add/1` delete-and-retry recursion (potential infinite loop under a race).** If, between `start_child` returning `{:error, :already_present}` and the `delete_child` call, another process restarts the child so `delete_child` returns `{:error, :running}` (ignored via `_ =`), the retry re-observes `:already_present` and could loop. This requires concurrent callers of an internal API, and both the `backend` value and the call timing are operator/developer-controlled — no attacker-forwarded data reaches it. A robustness/availability nit, not a security defect under this threat model.
- **`max_restarts: 30, max_seconds: 3` (restart-intensity as a crash-cascade lever).** An attacker who can make a *backend* crash on hostile log content could in principle burn this restart budget and tear down the backend supervisor. But the crash-on-content logic is in the backend/handler/formatter files, not here; this file only picks a standard (indeed generous) OTP restart intensity. Any finding belongs to the crashing component, not to this supervisor. Also note this is the deprecated legacy path (active only when `:backends` config is set); the default path uses the `:logger` handler directly.
- **`{:error, :already_present}` branch as dead defensive code.** Reachable: add a backend, let it terminate normally (leaving a "present" but non-running spec), then re-add → `:already_present`. Not dead; functions as an idempotent self-heal.

