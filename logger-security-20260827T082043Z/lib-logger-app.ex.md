# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/app.ex
<!-- sha256: a8f7712b00614c7a3271f384f05e48b4165c2a2098e4631b5ee8d086b1d8f299 -->

# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/app.ex
<!-- sha256: a8f7712b00614c7a3271f384f05e48b4165c2a2098e4631b5ee8d086b1d8f299 -->

_Re-reviewed on 2026-08-27_

Confirmed by execution: `inspect/1` escapes `\n`, `\r`, and `\e` into literal backslash sequences, so even a hostile `error` value on line 158 cannot smuggle newlines or ANSI. Every sink in this file consumes only operator config or internal `:logger` state — no attacker-forwarded data (log payloads, network bytes, files) reaches this module; that processing lives in `formatter.ex`/`utils.ex`/`translator.ex`.

## Summary
`Logger.App` is the OTP application callback that wires up `:logger` primary filters and the default handler from operator configuration and internal `:logger` state; no attacker-forwarded data reaches any sink in this file, and no security issue was found.

## Security
No findings.

**Inventory traced (all judged safe):**

| Line | Sink | Class | Source | Verdict |
|------|------|-------|--------|---------|
| 55 | `:logger.set_primary_config(:level, …)` | Shared state | `Application.get_env(:logger, :level)` (operator config) | trusted config |
| 46 | `Application.put_env(:logger, :default_formatter, …)` | Shared mutable state | operator `:console` config | trusted config, logger's own namespace |
| 60 | `:logger.set_primary_config(:metadata, Enum.into(metadata, …))` | Resource consumption | `Application.fetch_env!(:logger, :metadata)` | operator config; huge value is a footgun, not attacker-reachable |
| 63–68 | `add_primary_filter` w/ `translators` config | Protocol/behaviour dispatch | `Application.fetch_env!(:logger, :translators)` | operator-supplied `{mod,fun}`; trusted. Filter *logic* is in `utils.ex`, not here |
| 153 | `:logger.add_handler(:default, config.module, config)` | Protocol/behaviour dispatch | `config.module` from operator `:default_handler` config (default `:logger_std_h`) | operator-controlled module dispatch; trusted |
| 158 | `IO.puts(:stderr, "…: #{inspect(error)}")` | Template/interpolation | `error` from `:logger.add_handler` return (internal) | `inspect` escapes CR/LF/ANSI (executed, see above); value not attacker-controlled |
| 167 | `apply(:logger, fun, args)` | Code execution / dispatch | `fun` ∈ `{:set_primary_config, :add_handler, :remove_handler}` (all literal); `args` = internal `:logger` state | `fun` never externally derived; trusted args |

## Risks / caveats
- **Attacker model fit.** The realistic adversary for this folder is a remote peer whose data gets *logged*. That data is processed by the formatter/translator/utils modules — none of which are called here. This file only runs at application start/stop/config-change and only reads `:logger` app-env plus internal `:logger` state, so there is no path from attacker-forwarded bytes to any sink in `app.ex`. This matches the folder threat model's classification of operator configuration as trusted (misconfiguration = DoS footgun, out of the privilege boundary).
- **Config provenance (assumption removed).** The verdicts above rest on `:logger` app-env being operator/build-trusted, which the standing folder threat model states explicitly ("Operator configuration … Trusted, but misconfiguration … is a DoS surface"). This conclusion does **not** depend on any assumption about whether transitive dependencies can populate `Application` env: even a dependency that *could* set `:logger` `:level`/`:metadata`/`:default_handler` would thereby be exercising a capability strictly weaker than the arbitrary compile-time code every dependency already runs at build time — so it is not a privilege escalation under any provenance model. No residual assumption remains.
- **Verified vs. asserted.** I verified (executed) that `inspect/1` escapes control characters (line 158) on the OTP build under review. The remaining judgments are trust-boundary/data-flow arguments over a small, fully-enumerated set of config sources; there is no exploit behavior to reproduce because no sink here is attacker-reachable.

### Ruled out
- **Log/terminal injection at line 158** — `error` comes from `:logger.add_handler`'s return, not attacker input, and is passed through `inspect`, which escapes `\n`/`\r`/`\e` (executed: `{:error, "boom\n\e[2Jclear"}` → `"{:error, \"boom\\n\\e[2Jclear\"}"`). No newline or ANSI smuggling. *Axes examined: control-char escaping (CR/LF/ESC) via executed `inspect`; source provenance (internal `:logger` return). | Not examined: none.*
- **Dynamic dispatch `apply(:logger, fun, args)` (167)** — `fun` is only ever the literal atoms `:set_primary_config`/`:add_handler`/`:remove_handler` constructed in this file (lines 70, 136, 155); `args` carry internal `:logger` state (`primary_config`, handler `module`/`config`). No external influence on the callee or its arguments. *Axes examined: callee-atom provenance (all literal, in-file); arg provenance (internal `:logger` state). | Not examined: none.*
- **Operator-controlled handler `:module` reaching `:logger.add_handler` (153)** — module name comes from `:default_handler` config; operator/build-trust, not network/attacker-reachable. *Axes examined: source provenance (operator app-env). | Not examined: none.*
- **`Application.put_env(:default_formatter)` shared-state write (46)** — value from operator `:console` config, into logger's own env namespace, during single-threaded application start; no cross-tenant or attacker path, and the `with :error <-` check-then-act is not attacker-drivable (application-controller-serialized startup). *Axes examined: source provenance; concurrency (app-controller serialized start). | Not examined: none.*
- **Metadata merge memory growth (60)** — bounded only by operator-supplied `:metadata`; a DoS footgun for the operator, not something a remote attacker can influence through a documented public API. *Axes examined: source provenance (operator config). | Not examined: none.*
- **Atom creation / `Code.eval` / `File`/`Path` / `binary_to_term` / `Module.concat` / command exec** — none present (grep-confirmed); every atom in the file is a literal.

### Did I verify this?
- **Executed:** `inspect/1` control-character escaping (CR/LF/ESC) on the OTP build under review — output cited above.
- **Read (this pass):** the full file (`app.ex`, 170 lines) to re-confirm that `fun`/`args` at line 167 originate only from in-file literals (70/136/155) and internal `:logger` state, and that no atom/eval/path/deserialization sink exists.
- **Argued, not executed:** the trust-boundary judgment that operator app-env is trusted — grounded in the standing folder threat model, and made provenance-independent (see *Config provenance* above) so it no longer rests on any assumption. There is no attacker-reachable sink in this file to reproduce.

### Dropped in re-review
None. All prior findings-of-safety (the "No findings" verdict and each ruled-out entry) are preserved; the only change is the removal of the "I assume" hedge in the Config provenance caveat, replaced by a provenance-independent argument, plus executed evidence for the line-158 `inspect` claim.
