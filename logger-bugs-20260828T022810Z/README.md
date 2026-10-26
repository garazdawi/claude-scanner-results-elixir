# Logger Bugfix Scan — Ranked Summary

**Scope:** Elixir `Logger` (`lib/logger/`). No `VALIDATION.md` was present, so findings are ranked by severity and confidence from `CROSS_FILE.md` and the per-file reports. Both `CLUSTER_*.md` files report **no cluster findings**. Reviews were re-run with executed (not merely asserted) evidence against the source under review.

---

### 1. Stopping `:logger` leaks the Elixir default handler and loses the Erlang one — **High**
`lib/logger/app.ex:154-155` · `CSCAN-app-remove-handler-list-wrapped` · confidence high, **verified**

`start/2` builds its revert state as a list of `{fun, args}` action tuples that `redo/1` replays on stop. But on successful install, `add_elixir_handler/1` returns the removal action *wrapped in an extra list* (`[{:remove_handler, [:default]}]`). That list element fails the `{fun, args}` pattern in `redo/1`'s comprehension and is **silently dropped**. Consequently, `Application.stop(:logger)` never removes the Elixir `:default` handler, the follow-up re-add fails with `{:already_exist, :default}`, and the original Erlang default handler is never restored. Verified end-to-end: after stop, `:default` is still the Elixir handler. This corrupts global logging state across stop/restart cycles (test suites, releases, config reloads) and **compounds** on each cycle, since the next start captures the leftover Elixir handler as if it were the Erlang default. Fix: return the bare tuple `{:remove_handler, [:default]}`.

### 2. `changing_config/3` is non-total — non-`:refresh` reconfiguration of the Logger handler crashes — **Medium**
`lib/logger/backends/handler.ex:19` · `CSCAN-handler-changing-config-nontotal` · confidence high, **verified**

The `:logger_handler` behaviour invokes `changing_config/3` for *every* config change, but this module implements only the single `changing_config(:update, config, %{config: :refresh})` clause. Any documented non-refresh OTP call — e.g. `:logger.set_handler_config(Logger, :level, :info)` or `:logger.update_handler_config(Logger, :filters, [])` — matches no clause and raises `FunctionClauseError`, which OTP wraps as `{:error, {:callback_crashed, ...}}`, leaving the change unapplied. Elixir's own Logger only ever calls `:refresh` (works), so this bites callers who manage OTP handlers uniformly. Fix: add a catch-all `changing_config(_op, _old, new), do: {:ok, new}`.

### 3. `prune/1` fails to strip UTF-16 surrogates — "sanitized" data is still unencodable — **Medium**
`lib/logger/formatter.ex:359` · `CSCAN-formatter-prune-allows-surrogates` · confidence high, **verified**

`prune/1` is documented to remove invalid Unicode code points, but its list-clause guard `h in 0..1_114_111` admits the entire surrogate range `0xD800..0xDFFF`, which is invalid in UTF-8. A surrogate passes through untouched, so `prune/1`'s own return value still fails `:unicode.characters_to_binary/1` — the function's whole purpose is defeated. Reachable with peer/attacker-influenced code-point lists via the documented public API. Boundary sweep confirms the whole surrogate block survives. Fix: exclude `0xD800..0xDFFF` in the guard so surrogates fall through to the replacement clause.

---

### Lower-severity / localized findings

### 4. `compile/1` raises on a format string that *begins* with a literal `$` — **Low**
`lib/logger/formatter.ex:442-448` · `CSCAN-formatter-compile-leading-dollar-literal` · verified. A leading `$` not followed by `[a-z]+` (e.g. `compile("$5 today")`) is mis-parsed as a `$name` pattern, raising a confusing `ArgumentError` naming the whole string; the same `$5` mid-string is correctly treated as a literal. Compile-time operator config only, hence Low.

### 5. `system_time_to_date_time_ms/2` returns an out-of-contract negative ms for pre-epoch timestamps — **Low**
`lib/logger/formatter.ex:396,404` · `CSCAN-formatter-negative-systime-ms` · verified. A negative `system_time` yields a negative remainder, so the ms field violates the declared `0..999` type and renders garbage (`23:59:59.00-500`). The live `format/2` path guards `time >= 0`, so only custom-formatter authors calling the helper directly are affected. Fix: use `Integer.mod/2`.

### 6. `format_event/2` lets an arbitrary `report_cb` (or raising `Inspect`) escape the handler's containment — **Low**
`lib/logger/backends/handler.ex:49` + `formatter.ex:291-302` · `CSCAN-handler-format-event-uncaught-raise` / `CSCAN-crossfile-formatter-format-event-nontotal` · confidence low, **verified**

Cross-file finding (surfaced only from the handler's read into the formatter; the per-file formatter review missed it — the two reports **contradict** on whether `format_event/2` is total). `log/2` guards only `rescue ArgumentError` / `catch :exit`, but `format_message({:report, data}, %{report_cb: cb}, _)` invokes an arbitrary caller-supplied `cb.(data)` that can raise any class or `throw`, escaping the guard and crashing the handler callback. OTP's `:logger` core contains the crash (event loss / possible handler removal, not a VM crash), hence Low. Fix belongs in `formatter.ex` (keep `format_event/2` within its `IO.chardata()` contract) or broaden containment in `handler.ex`.

### 7. Moduledoc typo: `:warn` maps to `:warnings` — **Low (doc)**
`lib/logger.ex:499` · `CSCAN-logger-warn-maps-to-warnings-typo` · verified. The docs claim `:warn` maps to `:warnings`, a level that does not exist; the real mapping (handler.ex:125-134) is `:warning`. Documentation only.

---

### Unresolved cross-file caveat (no finding, worth a follow-up)
`CSCAN-crossfile-warn-keyerror-erl-level-invariant-unverified` (**medium confidence, Unresolved**): both `console.ex:252` (`color_event`) and `formatter.ex` (`colorize/1`) dismiss a `Map.fetch!(colors, level)` `KeyError` on the legacy `:warn` level by relying on `handler.ex:58` *unconditionally* injecting `erl_level` with an OTP level — but **no in-folder report actually verifies that injection is unconditional**. If any documented path lets a caller suppress/overwrite `erl_level`, both dismissals collapse into live `KeyError` crashes. Recommend verifying the invariant in `handler.ex` source and recording it explicitly, or adding a defensive `Map.get`/catch-all for the `:warn` key.

---

### Files with no surviving findings
`translator.ex`, `backends/config.ex`, `backends/console.ex`, `backends/internal.ex`, `backends/supervisor.ex`, `backends/watcher.ex`, `utils.ex`, `mix.exs` — all reviewed with executed evidence and cleared (translator.ex is byte-identical to upstream; config.ex's shared-counter and sole-handler cross-file invariants were confirmed by reading co-defining lines).
