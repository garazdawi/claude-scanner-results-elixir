# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/config.ex
<!-- sha256: f34133ca953616fe9ae04456af8246fe13e5c61f910adbe5ad5a4d57f9934e79 -->

_Analysed on 2026-08-27T08:28:57Z_

I've read the file in full and traced the one real value-flow of interest — the `Application.put_env` sink in `handle_call` — back through its only in-repo caller. The path is `Logger.Backends.Internal.configure/1` → `Keyword.take(options, [:sync_threshold, :discard_threshold, :truncate, :utc_log])` → `Logger.Backends.Config.configure/1`, so keys are allow-listed and values are operator-supplied logger config. No network/disk/dependency data reaches any sink here.

Here is the report.

---

## Summary

`Logger.Backends.Config` is a tiny legacy `:gen_event` overload-protection/config handler whose only state-mutating sink (`Application.put_env/3`) is fed exclusively by allow-listed operator configuration, and it contains no reachable untrusted-data path — no findings.

## Security

No findings.

Sink inventory and disposition (Phase 1 → Phase 2):

- **`handle_call/2` → `Application.put_env(:logger, key, value)`** (config.ex:34–37) — *Shared mutable state*. Value origin: the `options` keyword list passed to `configure/1` (config.ex:12). The only in-repo caller is `Logger.Backends.Internal.configure/1` (internal.ex:39), which narrows the list with `Keyword.take(options, [:sync_threshold, :discard_threshold, :truncate, :utc_log])` before it arrives, so both keys and shape are operator-controlled config, not forwarded network/disk/dependency data. No atom is created here (keys are already atoms; there is no `String.to_atom`). Out of scope: operator config is trusted per the folder threat model and the "library caller is not the attacker" rule. Not a finding.
- **`warn/1` interpolation** (config.ex:84, 90) — *Template/interpolation*. The only interpolated value is `total`, an integer read from `:counters.get/2` (config.ex:74). Integers carry no injectable payload; the rest of the string is a compile-time literal. Safe.
- **`schedule_update_counter/1` → `Process.send_after(self(), …, discard_period)`** (config.ex:107) — *Resource consumption / robustness*. `discard_period` = `Application.fetch_env!(:logger, :discard_threshold_periodic_check)` (operator config). A bad value is self-inflicted misconfiguration, not attacker-reachable. Not a finding.
- **`update_counter/2` counter/message-queue accounting** (config.ex:62–96) — *Concurrency / resource consumption*. This *is* the log-flood mitigation (bounds work at `:discard_threshold`), not a sink. The read-counter-then-add ordering and the "lost bumps" TOCTOU are explicitly documented and benign. Not a finding.
- **`configure/1` → `:gen_event.call(Logger, @name, …)` and `warn/1` → `:gen_event.notify(self(), …)`** (config.ex:13, 103) — *Concurrency / messaging*. In-VM message passing only; reaching it requires a co-located BEAM process, which the folder threat model places out of scope. Not a finding.

No code-execution, compile-time-execution, atom-creation-from-bytes, deserialization, path/file/command, archive, crypto, network, regex/ReDoS, or Unicode sinks are present in this file.

## Risks / caveats

- My "operator config only" conclusion rests on the observed call graph: the sole in-repo route into `configure/1` applies a 4-key allow-list (`internal.ex:39`). A downstream/out-of-tree caller that invoked `Logger.Backends.Config.configure/1` directly with attacker-forwarded key/value pairs would write arbitrary `:logger` app-env entries — but that call is undocumented (`@moduledoc false`) and reaching it requires an in-VM sender, so it stays out of scope. No behavioral reproduction was needed because the conclusion is a negative (no untrusted-data path reaches a sink); the evidence is the static call-chain grounding above rather than a runnable exploit.

### Ruled out

- **`Application.put_env` config poisoning** — keys/values are operator configuration (allow-listed to `:sync_threshold, :discard_threshold, :truncate, :utc_log` on the real path); not data forwarded from network/disk/dependency/build, so it fails the scope test even though `Application.put_env/3` is on the checklist.
- **Atom exhaustion** — no `String.to_atom`/`binary_to_atom`/`Module.concat` anywhere; `put_env` keys are pre-existing atoms.
- **Log injection via `warn/1`** — only an integer is interpolated; no attacker-controlled string reaches the message.
- **`Process.send_after` crash on bad period** — operator-config-only, self-inflicted, not attacker-reachable.
- **Counter TOCTOU / message-queue race** — documented, intentional, and the drift is self-correcting via the periodic check; it is the DoS mitigation, not a vulnerability.

