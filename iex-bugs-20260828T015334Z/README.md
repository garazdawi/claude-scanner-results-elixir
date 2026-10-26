# IEx Bugfix Scan — Ranked Findings

Source tree: `elixir/lib/iex` (Elixir 1.21.0-dev, tree `3389f8405`). No `VALIDATION.md` was present, so ranking is driven by `CROSS_FILE.md`, the two `CLUSTER_*` reports, and the per-file reports. Almost every finding below is backed by `verified` (executed) evidence.

---

## 1. Systemic: a bad argument to a documented API wipes shared state for every shell in the session — **High**
*Cross-file finding ([`CSCAN-crossfile-singleton-genserver-raise-wipes-shared-state`](./CROSS_FILE.md#cscan-crossfile-singleton-genserver-raise-wipes-shared-state)); files: `config.ex`, `pry.ex`, `broker.ex`. THREAT_MODEL §Supervisor.*

Three of the four singleton supervised processes share one defect shape: fallible validation/reflection runs **inside** the state-owning process's own callback, so an ordinary bad argument raises there instead of returning an error to the caller. The supervisor restarts the process with fresh state, silently destroying the shared state it owned for the whole session. This is the highest-impact item because no single-file review could name it as systemic, and all three constituents are individually verified:

- **`pry.ex` (High) — [`CSCAN-pry-remove-breaks-undef-module-crash`](./lib-iex-pry.ex.md#cscan-pry-remove-breaks-undef-module-crash).** `IEx.Pry.remove_breaks(SomeModule)` on an unloadable/mistyped module calls `module.__info__(:attributes)` in `instrumented?/1`, raising `UndefinedFunctionError` inside the `IEx.Pry` GenServer instead of returning the documented `{:error, :no_beam_file}`. The crash destroys the `IEx.Pry` ETS table — **every breakpoint in the session is lost**. Fix: guard the reflective lookup (`Code.ensure_loaded?/1` or `try/rescue`).
- **`config.ex` (Medium) — [`CSCAN-config-invalid-option-crashes-agent`](./lib-iex-config.ex.md#cscan-config-invalid-option-crashes-agent).** A typo'd option in `IEx.configure/1` (e.g. `color:` for `colors:`, or `width: 1.5`) runs `validate_option/1`'s `raise ArgumentError` inside the shared `IEx.Config` agent, crashing it and wiping the `:after_spawn` hook table; the caller sees an opaque `exit`, not the intended `ArgumentError`. Fix: validate in the caller before `Agent.update`.
- **`broker.ex` (Medium) — [`CSCAN-broker-erpc-remote-crash`](./lib-iex-broker.ex.md#cscan-broker-erpc-remote-crash).** In a cluster, a peer that fails `:erpc.call(node, IEx.Broker, :shell, [])` (plain Erlang node → `:undef`, or `:noconnection` mid-scan) re-raises inside `servers/1`, crashing the `IEx.Broker` GenServer during a `take_over/3`. All shells lose their takeover registration and the `:infinity` caller hangs/exits. Distributed Erlang is an explicit trust boundary. Fix: wrap the remote call in `try/rescue`.

Recommended remediation is one folder-level hardening pass: move fallible validation/reflection to the caller, or wrap the in-callback call to return the documented error tuple. (`MixListener`, the fourth singleton, was audited and does **not** exhibit this pattern.)

---

## 2. `broker.ex` spec violation crashes autocomplete via `send(nil, …)` — **Medium**
*Cluster finding ([`CSCAN-cluster-broker-evaluator-nil-pid`](./CLUSTER_lib-iex-0.md#cscan-cluster-broker-evaluator-nil-pid)); files: `broker.ex`, `autocomplete.ex`, `evaluator.ex`.*

`broker.ex:36` specs `evaluator/1` as `{pid, pid} | nil`, but the body returns `{dictionary[:evaluator], pid}` unconditionally. During a shell rerun/respawn there is a window (server.ex sets/deletes the `:evaluator` key) where `dictionary[:evaluator]` is `nil`, so it returns `{nil, pid}`. Autocomplete's `with {evaluator, server} <- IEx.Broker.evaluator(shell)` still matches that 2-tuple, binds `evaluator = nil`, and passes it into `IEx.Evaluator.fields_from_env/value_from_binding/variables_from_binding`, whose first act is `send(nil, …)` → `ArgumentError`, propagating out of the un-`try`'d non-remsh `expand/2` path. (A dead pid also makes the `{:dictionary, _} = Process.info(...)` match raise `MatchError`.) Fix: make `evaluator/1` honor its contract — guard the pid and dict read and return `nil` so autocomplete falls through to its empty-result branch. Rated Medium confidence by the cluster reviewer.

---

## 3. `helpers.ex` — `process_info/1` crashes on a name-based monitor — **Medium**
*[`CSCAN-helpers-process-info-name-monitor-crash`](./lib-iex-helpers.ex.md#cscan-helpers-process-info-name-monitor-crash) (verified).*

`IEx.Helpers.process_info/1` on any live process that monitors another **by registered name** aborts mid-print with `FunctionClauseError` in `pid_or_port_details/1`, which has no clause for the `{name, node}` tuple that `:erlang.process_info(pid, :monitors)` returns for such monitors. Monitoring by name is ordinary OTP practice (normal GenServers/supervisors), so this is readily hit; output is truncated at the "Monitors" pane. Fix: add a `{name, node}` clause (and a `nil` fallback for `to_process_details/1`).

## 4. `iex.ex` — `break!/2` over-escapes its `stops` argument — **Medium**
*[`CSCAN-iex-break-stops-macro-escape`](./lib-iex.ex.md#cscan-iex-break-stops-macro-escape) (verified).*

`iex.ex:766` uses `unquote(Macro.escape(stops))`, so any non-literal stop count (`break! URI.parse/1, count`, `@default_stops`, `2+3`) is stored as an **AST tuple** instead of an integer. `breaks()` reports garbage, and the first time the instrumented function fires, `:ets.update_counter/3` gets a tuple in the counter slot and raises `ArgumentError` — the pried function blows up instead of pausing. Only hint is a spurious "unused variable" warning. Fix: `unquote(stops)` (identical for literals, so no regression).

---

## 5. Lower-impact findings

- **`broker.ex` — [`CSCAN-broker-shell-monitor-leak`](./lib-iex-broker.ex.md#cscan-broker-shell-monitor-leak) (Low, verified).** Every `take_over/3` → `servers/1` does `Process.monitor(pid)` per reachable shell but never `Process.demonitor`s on the accept/refuse/DOWN paths, leaking one monitor per shell per takeover cycle (plus a redundant `:DOWN` each when a shell dies). Fix: demonitor the transient shell refs when a takeover resolves.
- **`info.ex` — [`CSCAN-info-code-which-empty-charlist`](./lib-iex-info.ex.md#cscan-info-code-which-empty-charlist) (Low, cosmetic, verified).** `i(Mod)` on a memory-loaded module (defined at the prompt / `Code.compile_string`) prints a meaningless `"[]"` for "Module bytecode": `:code.which/1` returns `[]`, which the atom-intended catch-all renders via `inspect/1`. Fix: add an explicit `[] ->` clause (e.g. `"in memory"`).
- **`server.ex` — [`CSCAN-server-run-opts-missing-dot-iex`](./lib-iex-server.ex.md#cscan-server-run-opts-missing-dot-iex) (Low, spec/doc mismatch, `asserted`).** `run/1`'s `@doc` documents `:dot_iex` and `init_state/1` reads it, but the `run_opts` type behind `run/1`'s `@spec` omits it, so a documented `run(dot_iex: …)` call is a Dialyzer type error. Runtime is unaffected. Fix: add `dot_iex: String.t()` to the type.
- **`introspection.ex` — [`CSCAN-introspection-t1-leaks-private-types`](./lib-iex-introspection.ex.md#cscan-introspection-t1-leaks-private-types) (Low, low confidence).** `t(Module)` prints `@typep` (private) type definitions, while `t({mod, type})` / `t({mod, type, arity})` deliberately refuse them (`kind in [:type, :opaque]`) — an inconsistency/minor disclosure. The reviewer could not confirm against upstream whether `t/1`'s behavior is intended; downgrade if so.

---

## Files with no surviving findings
`autocomplete.ex` (source of the cluster crash but clean standalone), `evaluator.ex`, `history.ex`, `mix_listener.ex`, `app.ex`, `mix.exs` — reviewed with no reportable defects.

## Cross-file coverage caveats (from CROSS_FILE.md ledger)
No **Contradicted** dependencies. Recorded **Unresolved** gaps (not promoted to findings): external module contracts (`Code.Fragment.*`, `:elixir.string_to_tokens`); the `Broker` reply-once invariant under concurrent multi-server interleavings (asserted, not executed); and the `broker.ex` `shell/0`/`evaluator/1` **TOCTOU race** where a pid dies between the liveness check and `Process.info` (untried by any report). These are candidates for follow-up if deeper assurance on the broker/takeover path is wanted.
