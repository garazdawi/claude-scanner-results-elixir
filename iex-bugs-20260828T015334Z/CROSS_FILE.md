# Cross-file findings

The per-file reports do not use the explicit `* Depends on:` line format (the grep returns nothing). The ledger below is therefore reconstructed from the **implicit cross-module contracts** each report names in its `Risks / caveats` and `Ruled out` sections, plus the cross-file invariants enumerated in `THREAT_MODEL.md`. Every such asserted dependency is accounted for.

## Asserted-dependency ledger

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| autocomplete.ex | `IEx.Broker.evaluator/1` returns documented pid/nil shape | Confirmed | broker.ex report examines `evaluator/1`, verifies the `{:dictionary,_}` match and the `if pid` nil-guard. |
| autocomplete.ex | `IEx.Evaluator.value_from_binding`/`variables_from_binding`/`fields_from_env` honor `{:ok, any} \| :error` | Confirmed | evaluator.ex "Ruled out" verified these return the spec'd `{:ok,_}\|:error`. |
| autocomplete.ex | `Code.Fragment.cursor_context`/`container_cursor_to_quoted` return shapes | Unresolved | External module; no report in this set covers it. |
| autocomplete.ex | remsh `:erpc.call` on `:shell.whereis()` pid is in-contract | Unresolved | Explicitly treated as in-contract, not audited as a trust boundary anywhere. |
| evaluator.ex | `:elixir.string_to_tokens`/`tokens_to_quoted` error shapes (OTP-29/1.21) | Unresolved | External tokenizer; confirmed only against one build, not covered by another report. |
| evaluator.ex | `.iex.exs` eval-from-cwd is operator-accepted risk | Confirmed | THREAT_MODEL §Trust boundaries documents this as accepted operator risk. |
| THREAT_MODEL | `Server.__parse__/3` is the sole catcher of `Evaluator.parse/3` raises | Confirmed | server.ex ruled-out "`__parse__/3` get_until threading" verifies clause-3 `catch` surfaces `{:error,…}`. |
| THREAT_MODEL | `Config.validate_option/1` rejects bad config before readers dereference | Confirmed (caveat) | config.ex verifies readers (`width/0`,`parser/0`,`merge_option`) are safe; but see systemic finding — the reject path itself crashes. |
| THREAT_MODEL | `Broker` reply-once invariant spans accept/refuse/DOWN | Unresolved | broker.ex rules this out `asserted` (source-trace) **not verified**; concurrent multi-server interleavings untested by any report. |
| THREAT_MODEL | `Evaluator` holds group leader + `:iex_server`/`:iex_evaluator` PD keys; pry restores | Confirmed | evaluator.ex ruled-out the re-entrant `init/5`/`loop/1` signaling as intentional. |
| THREAT_MODEL | `Server` routes `:take_over` by `opts[:evaluator]` pid uniquely | Confirmed | server.ex ruled-out `take_over?` arity dispatch and routing across the three cond outcomes. |
| broker.ex | `shell/0`/`evaluator/1` pid stays live between liveness check and `Process.info` (TOCTOU) | Unresolved | broker.ex flags this race as untried; autocomplete relies on `evaluator/1` and does not cover it either. |
| mix.exs | OTP `env` defaults (`history_size: 20`, prompts, `auto_reload: false`) agree with `IEx.Config` keys | Unresolved | config.ex report does not enumerate these default keys/values; agreement judged only against THREAT_MODEL prose. |
| mix_listener.ex | `IEx.Config.auto_reload?()` reader | Unresolved | config.ex report does not cover the `auto_reload?` reader path. |
| mix_listener.ex | `Mix.Task.Compiler.notify_modules_compiled` protocol (`os_pid`, `modules_diff`) | Confirmed | Verified against the byte-identical installed emitter at `/opt/elixir` (external but checked). |

No **Contradicted** rows: no report asserts a contract that the cited file's own report denies. The Unresolved rows are coverage gaps (external modules, an untested race, an asserted-only invariant) — recorded here, not promoted to findings.

## Findings

#### `CSCAN-crossfile-singleton-genserver-raise-wipes-shared-state`

- **Finding ID** — `CSCAN-crossfile-singleton-genserver-raise-wipes-shared-state`
- **Severity** — High
- **Confidence** — high
- **Files involved** — `lib-iex-config.ex.md`, `lib-iex-pry.ex.md`, `lib-iex-broker.ex.md` (context: `THREAT_MODEL.md` §Supervisor/process structure)
- **Issue** — Three of the folder's four singleton supervised processes share the *same verified defect shape*, which no single-file review could name as systemic: an ordinary bad argument to a **documented public API** raises *inside* the process's own `handle_call`/`Agent.update` callback rather than in the caller, crashing the singleton and silently destroying the shared state it owns for **every shell in the session**. Concretely: (1) `config.ex:164-212` — `IEx.configure/1` with a typo'd option runs `validate_option/1`'s `raise ArgumentError` inside the `IEx.Config` agent, wiping the `:after_spawn` ETS hook table (`CSCAN-config-invalid-option-crashes-agent`, verified); (2) `pry.ex:337→428→649` — `IEx.Pry.remove_breaks/1` on an unloadable module raises `UndefinedFunctionError` inside the `IEx.Pry` GenServer, destroying the `IEx.Pry` ETS breakpoint table (`CSCAN-pry-remove-breaks-undef-module-crash`, verified); (3) `broker.ex:167-174` — a failing peer `:erpc.call` raises inside the `IEx.Broker` `{:take_over,…}` `handle_call`, wiping `servers`/`takeovers` and hanging the `:infinity` caller (`CSCAN-broker-erpc-remote-crash`, verified). THREAT_MODEL §Supervisor already warns "Crashing `Broker` or `Config` takes out live shells' takeover/config; crashing `Pry` loses all breakpoints" — the three findings are the same warning realized through three documented entry points, each on trivially-reachable input. The systemic root cause is architectural: validation/reflection that can raise is performed inside the state-owning process instead of in the caller, so the supervisor's restart converts a would-be error return into total shared-state loss.
- **Resolution** — Treat as one folder-level hardening item: move all fallible validation/reflection to the caller process (config's own suggested fix: `Enum.each(options, &validate_option/1)` before `Agent.update`) or wrap the in-callback fallible call in `try/rescue` returning the documented error tuple (pry's and broker's suggested fixes). No further source reading needed — all three underlying findings are already `verified`; the only cross-file work is recognizing and fixing the shared pattern together (and auditing the fourth singleton, `MixListener`, whose `purge/0` `:code.purge` path does not exhibit it).
