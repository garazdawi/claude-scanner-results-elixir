# Security Scan Summary — Elixir `Logger` (`lib/logger`)

**Scope:** 12 source files (`logger.ex`, `formatter.ex`, `utils.ex`, `translator.ex`, `backends/*`, `app.ex`, `mix.exs`). No `VALIDATION.md` was produced, so verdicts below rest on the CROSS_FILE / CLUSTER reports and the per-file reviews directly.

**Bottom line:** This is a mature, well-defended module. Eight of twelve files have **no findings** (`logger.ex`, `utils.ex`, `handler.ex`, `internal.ex`, `supervisor.ex`, `config.ex`, `app.ex`, `mix.exs` — the content-forwarding and operator-config plumbing is all clean). **Every surviving finding is Low severity.** Only one is high-confidence. The residual issues cluster into two themes: (a) a cross-file lifecycle DoS in the deprecated `:gen_event` backend path, and (b) systemic non-sanitization of control characters (log/terminal-escape injection), which the whole stack defers to a formatter that declines to sanitize by design.

---

## Ranked findings

### 1. Orphaned `:logger` handler on backend crash-restart → Logger-app DoS + silent event drop `[CLUSTER, Low / medium-conf]`
`CSCAN-cluster-reinit-orphaned-logger-handler` — `internal.ex` + `handler.ex` + `config.ex` + `supervisor.ex` + `watcher.ex`

`Logger.Backends.Internal.init/1` installs a **global, process-less** `:logger` handler (`internal.ex:151-157`) but has **no `terminate/2`** and nothing removes it on a subtree restart — the only remover (`app.ex:91`) fires solely on app *stop*. A legacy backend that crash-loops on hostile logged content burns the backends-supervisor restart budget (`max_restarts: 30/3s`) → `rest_for_one` propagates → `Internal` restarts → `init/1` re-runs → `:logger.add_handler` now returns `{:error, {:already_exist, Logger}}` → **MatchError before any child starts** → `Logger.Supervisor` (the Logger app root) is taken down. Compounding this, during the down-window the orphaned `:counters` reference keeps incrementing with its resetter (`config.ex`) dead, so `value >= discard` becomes permanently true and `log/2` **silently discards every event**.

**Why it leads:** largest blast radius (whole Logger subtree + a stealthy "logging goes blind" audit-loss window), and it is a genuine cross-file finding no single-file reviewer could surface — the `internal.ex` per-file pass correctly saw `add_handler` succeed once and reported clean. **Caveats:** requires the deprecated `config :logger, :backends` path and a crash-prone custom backend; rated medium confidence. Fix: make `init/1` idempotent (`:logger.remove_handler(Logger)` before re-adding) and add a symmetric `terminate/2`.

### 2. `Logger.Formatter.compile/1` interns unvalidated atoms → atom-table exhaustion / VM abort `[formatter.ex, Low / HIGH-conf]`
`CSCAN-formatter-compile-unsafe-atom` — `formatter.ex:444`

Every distinct `$[a-z]+` fragment in a format pattern is converted to an atom via `String.to_unsafe_atom` **before** it is validated against `@valid_patterns` (validation happens afterward in `compile_code/1`). The atom is created and never GC'd **even on the raising path** (verified: 500 invalid fragments → 512 permanent atoms), so unbounded distinct fragments monotonically exhaust the atom table until the **entire BEAM node aborts**; a single fragment >255 chars raises `SystemLimitError` (CWE-770).

**Why #2:** the only **high-confidence, executed** finding in the scan, and its impact is node-wide (atom exhaustion kills the whole VM, not just Logger). Ranked below #1 only because reachability is narrower — it bites only if an application feeds attacker/dependency-influenced data as a *log format pattern* (`compile/1` / `new(format:)`), a documented-but-unusual "printf-style format-string injection" pattern; normal format patterns are operator/build config. Fix: validate the fragment as a string (`String.to_existing_atom` in a `try`, or membership check) before interning.

### 3. Systemic CR/LF/ANSI passthrough — log-line & terminal-escape injection `[CROSS_FILE, Low / medium-conf]`
`CSCAN-crossfile-control-char-passthrough-deferral` — spans `logger.ex`, `formatter.ex`, `console.ex`, `watcher.ex`, `translator.ex` (CWE-117 / CWE-150)

A responsibility-deferral pattern, not a single-file bug: attacker-influenced content carrying raw CR/LF/ANSI/CSI flows through the stack un-escaped. `logger.ex` forwards verbatim ("the formatter's job", verified), `console.ex` defers to the formatter, but `formatter.ex` states `prune`/`colorize` **deliberately do not** strip CR/LF or CSI "by design" — so the layer everyone points at declines ownership. The threat model endorses preserve-CR/LF as long-standing documented behavior (keeping it Low), but the finding records that **no layer sanitizes**, and it is confirmed reachable at two concrete, independently-filed sinks:

- **`watcher.ex:60` — `CSCAN-watcher-exit-reason-injection` (Low / med, verified end-to-end):** a crashing legacy backend's `Exception.format_exit/1` reason (which can embed attacker-logged data) is written **raw to stderr**. Reproduced forging a `[error] FORGED ADMIN ACTION` line + raw `\e[..m` bytes. Notably this stderr fallback path **bypasses the formatter entirely**, so the collective "formatter owns sanitization" defense doesn't even apply. Deprecated `:backends` path, gated behind the "logging is broken" fallback branch.
- **`translator.ex:96/208/701` — translator log injection (Low / low, verified):** `Exception.format_banner/3` and (newly audited) `Application.format_error/1` emit **raw** LF/ANSI when the exit reason is an exception (or `{exception, stacktrace}`) with an attacker-influenced message. Faithfully mirrors standard Elixir crash rendering; a fix here would diverge from `Kernel` behavior and belongs in the formatter.

### 4. `console.ex` unvalidated `:ansi_color` metadata → terminal-escape injection + handler-crash DoS `[console.ex, Low / low-conf]`
`CSCAN-console-ansi-color-metadata-unvalidated` — `console.ex:252-253`

If an application bulk-merges untrusted data into log metadata under the `:ansi_color` key, the value is passed straight to `IO.ANSI.format_fragment/2`. A **binary** value emits raw ANSI/OSC escapes to the operator terminal verbatim (window-title set, cursor moves, color spoofing — reproduced); an **atom** value that isn't a known color raises `ArgumentError` *inside* `handle_event/2`, killing the `:gen_event` handler and removing the console backend — repeated triggers exhaust the watcher restart loop and take the legacy backend subtree down (log loss). Lowest-ranked: low confidence, atypical reachability (requires the app to route untrusted data into `:ansi_color`), and the atom/DoS variant needs an atom value (raw network bytes aren't atoms), so the practical remote effect is the terminal-escape variant, which largely overlaps finding #3.

---

## Coverage notes & caveats

- **No Contradicted ledger rows.** The CROSS_FILE ledger resolved most cross-file dependencies as *Confirmed*. Five rows are **Unresolved** — the config.ex out-of-tree direct caller, the console.ex `erl_level`-always-valid invariant, the handler.ex `:logger`-validates-level invariant, the internal.ex/watcher.ex untrusted-backend-forwarding caveat, and the logger.ex handler-registry trust. Per the reviewing protocol these are **coverage gaps, not defects** (each concerns OTP `:logger` internals or arbitrary application callers outside the folder callgraph, or was verified by the source file's own execution); none is promoted to a finding. `CLUSTER_lib-logger.md` reported no findings.
- **Common threads:** every surviving issue lives on the **deprecated `:gen_event` `:backends` path** (#1, #3-watcher, #4) or requires an application to pass untrusted data into a **normally-trusted config position** (#2 format pattern, #4 metadata key) — i.e. caller-contract edges rather than default-path remote exploits. This is consistent with a hardened default handler path.
- **Operator-config sinks are uniformly out of scope** and were correctly dismissed across `app.ex`, `config.ex`, `internal.ex`, `supervisor.ex`, and `mix.exs` (all `Application.put_env`/`add_handler`/backend-module dispatch fed by operator/build-trusted config, per THREAT_MODEL.md).
- **No high-tier findings anywhere:** no RCE, no atom-creation-from-network-bytes on the default path, no deserialization/path/command sinks were found in any file.
