# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/internal.ex
<!-- sha256: 31e6ee8b9acdb301ece8e6f1d41c0903a69d789e8a74dc063eea99f496708513 -->

# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/internal.ex
<!-- sha256: 31e6ee8b9acdb301ece8e6f1d41c0903a69d789e8a74dc063eea99f496708513 -->

_Re-reviewed on 2026-08-27_

Supervisor/plumbing module for Elixir's legacy `:gen_event`-based Logger backend system. This pass resolves the two "outside this file" hedges from the first pass by reading the forwarding targets (`Logger.Backends.Supervisor`, `Logger.Backends.Config`, `Logger.Backends.Console`, `Logger.Backends.Watcher`).

## Phase 1 — Sink inventory

| # | Sink | Location | Input source |
|---|------|----------|--------------|
| 1 | `:gen_event.call(Logger, backend, {:configure, options})` — behaviour dispatch | `configure/2` (line 49) | caller-supplied `backend` + `options` |
| 2 | `Logger.Backends.Supervisor.add(backend)` — starts handler, invokes `init/1` | `add/2` (line 92) | caller-supplied `backend` module |
| 3 | `Logger.Backends.Supervisor.remove(backend)` | `remove/2` (line 117) | caller-supplied `backend` |
| 4 | `Keyword.take(options, @backend_options)` → `Logger.Backends.Config.configure/1` | `configure/1` (line 39) | caller options, allow-list filtered |
| 5 | `Process.whereis` + `Supervisor.start_child` — check-then-act | `ensure_started/0` (line 122–128) | internal |
| 6 | `Application.get_env`/`fetch_env!(:logger, …)` — shared config read | `init/1` (line 137–138) | operator app-env |
| 7 | `:logger.add_handler(...)` — installs primary handler | `init/1` (line 152) | internal + operator counter |

## Phase 2 — per-sink judgment (hedges now resolved)

- **Sink 1 forwarding path (was hedged: "handling defect belongs to those files").** Now verified by reading the two handlers this call can reach:
  - The `configure/1` (arity-1) allow-list path (sink 4) forwards to `Logger.Backends.Config.configure/1`, whose handler `config.ex:34-40` executes `Application.put_env(:logger, key, value)` **per option**. That is an unbounded app-env write keyed by `key` — but `internal.ex:39` runs `Keyword.take(options, @backend_options)` *before* the call, so `key` is confined to the fixed literal set `[:sync_threshold, :discard_threshold, :truncate, :utc_log]`. The `put_env` cannot be driven to an arbitrary/attacker-shaped key or atom. Confirmed no allow-list bypass and no atom pollution on this path.
  - The arity-2 `configure/2` (sink 1) does **not** allow-list; it forwards raw `options` to the *caller-chosen* `backend` handler (`console.ex:60`, `handle_call({:configure, options}, state)`). That handler is reached only via `Logger.configure_backend/2` with an operator-selected backend module — operator config, not attacker data. Reading `console.ex` confirms `internal.ex` adds no processing of its own; it is pure `:gen_event.call` plumbing. Any defect in option interpretation lives in `console.ex`/`config.ex`, and I have now confirmed those handlers exist and are the actual sinks — this is no longer an assumption.
- **Sinks 2–3 (`add`/`remove` → `Supervisor.add/remove`).** Read `supervisor.ex:11-37`: `add/1` builds a child spec `id: backend, start: {Watcher, :start_link, [{backend, backend}]}`, `restart: :transient`, and on `:already_present` deletes and retries; `Watcher.init/1` (`watcher.ex:15`) installs the backend via `:gen_event.add_sup_handler`. `backend` is a caller-supplied module atom the caller already holds — no atom is created here. This is arbitrary-module execution **by design** (documented `Logger.add_backend/2` contract), gated on operator/developer action. Confirmed not a data-flow from network/disk.
- **Sink 4.** Allow-list `Keyword.take` is a fixed literal; drops everything else before forwarding. Correct (see sink 1 note above for the downstream `put_env` confinement).
- **Sinks 5–7.** `ensure_started/0` check-then-act; `init/1` operator app-env + `:counters` handler install. Trusted operator config; only instantiated when deprecated `config :logger, :backends` is set.

No sink consumes attacker-influenced data through any documented public API. No decoder, path op, atom creation, or parser on external input exists in this file.

---

# Security review — `logger/backends/internal.ex`

## Summary
Operator-gated supervisor plumbing for the deprecated `:gen_event` Logger backend path. Every sink consumes trusted developer/operator configuration or the caller's own module choices. Both first-pass "outside this file" hedges are now verified against the forwarding targets: the `configure/1` allow-list confines the downstream `Application.put_env` in `config.ex` to four literal keys, and the option-handling logic genuinely resides in the `console.ex`/`config.ex` `handle_call` clauses this file only forwards to.

## Security
No findings.

## Risks / caveats
- **`backend` argument is arbitrary-module execution by design.** `add/2`, `remove/2`, `configure/2` dispatch on a caller-chosen module (`init/1` invocation via `Watcher`/`:gen_event.add_sup_handler`; `:gen_event.call`). Verified against `supervisor.ex:11-14` and `watcher.ex:15-20`: the module atom is passed straight through as a supervisor child id and `:gen_event` handler, with no atom creation in this file. This is the documented `Logger.add_backend/2` contract. It would only become a security issue if an application forwarded untrusted network/disk bytes as a backend module name — a *caller* contract violation. That misuse lives in arbitrary application code outside the `logger` folder callgraph and cannot be adjudicated from any neighbour module here; see "Did I verify this."

## Did I verify this?
- **Sink-1 forwarding (verified, not assumed):** read `config.ex:12-40` and `console.ex:37-60`. The arity-1 path's `put_env` keys are bounded by `internal.ex:35,39` `Keyword.take`; the arity-2 path forwards raw options to the caller's own backend `handle_call`. Both are the real sinks; `internal.ex` adds no processing. This clears the first-pass hedge "handling defect belongs to those files."
- **Backend-name execution (design contract, not a neighbour-verifiable claim):** the "downstream callers might forward untrusted bytes as a backend atom" hedge concerns arbitrary application callers of the public `Logger` API, which are not in this folder's callgraph. There is no neighbour module that could confirm or refute it, and this file itself never creates an atom or reads external data (grep-confirmed: no `String.to_atom`, `Module.concat`, `binary_to_atom`, decoder, or path op). It therefore cannot be upgraded to a finding *in this file* — there is no sink here that consumes external data. It remains a documented caveat about API misuse, not an unverified assumption affecting this file's verdict.

### Ruled out
- **`ensure_started/0` named-process race (TOCTOU).** Two concurrent callers can both observe `nil` and both call `Supervisor.start_child`; the loser receives `{:error, {:already_started, _}}`, which `ensure_started/0` discards (always returns `:ok`). No crash, no state corruption.
  Axes examined: control-flow of the discarded return value; supervisor idempotency. | Not examined: interleaving under real scheduler contention (not executed — no deterministic harness for the race; verdict rests on the return value being unconditionally discarded, which does not depend on timing). Entry: asserted.
- **`configure/2` `is_list(options)` guard → `FunctionClauseError`.** Reachable only by a caller violating the `@spec` (non-keyword). Contract violation by a trusted caller, not attacker data.
  Axes examined: guard reachability from public API. | Not examined: none (the only input carrying this axis is a caller-supplied non-list, which is a trusted-caller contract breach by definition; no attacker path exists).
- **Unbounded `add/2` backend creation (resource consumption).** Driven by trusted operator calls; `supervisor.ex:66` additionally bounds restarts (`max_restarts: 30, max_seconds: 3`). Confirmed by reading `supervisor.ex`.
  Axes examined: caller trust; restart-intensity bound. | Not examined: none.
- **`Application.get_env`/`fetch_env!` reads (shared state).** Operator-controlled app-env, trusted per threat model; no externally-derived key or value.
  Axes examined: input source. | Not examined: none.
- **Atom creation.** None in this file — no `String.to_atom`, `Module.concat`, or `binary_to_atom`; the backend atom is supplied already-interned by the caller. Grep-confirmed.
  Axes examined: static presence of atom-creating sinks. | Not examined: none.

### Dropped in re-review
None. Every first-pass judgment is preserved; the two hedges are resolved to verified statements (sink-1 forwarding) or a documented not-in-this-file caveat with the reason it cannot be a finding here (backend-name execution). Confidence in "No findings": **high** — raised from the first pass, since the forwarding targets were read and the allow-list confinement of the sole app-env write (`config.ex` `put_env`) is now confirmed rather than assumed.
