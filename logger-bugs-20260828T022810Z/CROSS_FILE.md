# Cross-file findings

## Asserted-dependency ledger

No report in this set uses the literal `* Depends on:` syntax; every cross-file dependency is expressed as a "Cross-file note", "cross-file invariant", hedged caveat, or an `asserted` dismissal that names another file. Each such entry is enumerated below.

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| THREAT_MODEL.md | `Utils.truncate_n/2` returns negative remainder exactly on overrun (relied on by `Formatter.truncate/2`) | Confirmed | utils.ex Ruled-out (line 18): negative-remainder-on-overrun contract "holds", verified. |
| THREAT_MODEL.md | `Translator.translate/4` honours the 4-value return contract (else-branch of `Utils.translator/2`) | Confirmed | translator.ex is byte-identical to upstream, adds no clause returning a foreign shape; utils.ex ruled-out covers exactly the four shapes. |
| THREAT_MODEL.md | `Backends.Config` is the sole always-present handler (Watcher stderr sentinel) | Confirmed | config.ex Ruled-out (lines 57-59) + watcher.ex Ruled-out (line 19) both confirm Config is the single persistent handler. |
| THREAT_MODEL.md | `Handler` & `Config` share `@counter_pos 1` on a size-1 atomics counter | Confirmed | config.ex Ruled-out (lines 27-33): handler.ex:80 and config.ex:60 both `@counter_pos 1`, internal.ex:139 creates size-1. |
| THREAT_MODEL.md | `Handler.log/2` and `Formatter.format/2` re-derive system_time with the same `time>=0` guard | Confirmed | handler.ex "Did I verify" (line 88) + formatter.ex finding (line 61) both show the `time >= 0` guard on the live path. |
| handler.ex | `Formatter.format_event/2` is total (won't raise past `rescue ArgumentError`) | **Contradicted** | handler.ex verified by execution it is NOT total (report_cb raise escapes); see finding below. |
| handler.ex | `Formatter.system_time_to_date_time_ms/2` benign for handler inputs | Confirmed | formatter.ex negative-systime finding (line 61) confirms the live path guards `time>=0`; handler passes only guarded non-negative. |
| handler.ex | Counter reconciliation lives in `Config.update_counter` (config.ex:62-96) | Confirmed | config.ex Ruled-out (lines 23-25) confirms Config is sole reconciler. |
| handler.ex | `gl` metadata always present via `:logger` do_log pipeline | Confirmed | handler.ex verified by execution via primary filter (OTP core, out of folder). |
| formatter.ex | `Utils.truncate_n/2`/`fix_binary/2` correctness (scope referral) | Confirmed | utils.ex Ruled-out (line 18) verifies truncation incl. surrogate/mid-codepoint edges. |
| formatter.ex | `colorize/1` KeyError unreachable because events carry OTP syslog levels | Unresolved | Rests on the level-injection invariant no in-folder report verifies; see systemic finding below. |
| console.ex | `handler.ex:58` always injects `erl_level` metadata carrying the OTP level | Unresolved | handler.ex report never verifies the `erl_level` injection is unconditional; see systemic finding below. |
| console.ex | `Config.warn/1` emits a `:warning`-level event (config.ex:102-103) | Confirmed | config.ex Ruled-out (line 41) confirms warn/1 broadcasts a `:warning` event. |
| internal.ex | `Supervisor.add/1` never returns `{:ok,_,_}` (add/2 case totality) | Confirmed | supervisor.ex Ruled-out (line 18): Watcher is a GenServer, no 3-tuple. |
| internal.ex | `{:error,:ignore}`/`{:error,reason}` init behavior realized in `Watcher.init/1` | Confirmed | watcher.ex Ruled-out (line 27) exercises the `{:error,:ignore}` fake-exit path. |
| internal.ex | `configure/1`'s `:ok = update_handler_config` holds because handler installed | Confirmed | handler.ex confirms the handler/`:refresh` path returns `:ok`. |
| supervisor.ex | `add/1` never returns 3-tuple because Watcher is a GenServer | Confirmed | Consistent with internal.ex reasoning + watcher.ex GenServer callbacks. |
| supervisor.ex | No supervisor leak on post-start raise (OTP link semantics) | Confirmed | OTP link invariant, out of folder; asserted and consistent. |
| config.ex | `Internal.configure/1` feeds a proper keyword list (Keyword.take) | Confirmed | config.ex read internal.ex:37-41; shape conforms. |
| config.ex | `discard_threshold`/`utc_log` env defaults present (Handler also fetch_env!s) | Confirmed | config.ex read handler.ex:27-36; shared Logger app-env defaults. |
| watcher.ex | `Config` always registered (init recursion termination + sentinel) | Confirmed | config.ex confirms Config is persistent. |
| logger.ex | `put_module_level`/`add_handlers` etc. delegate correctly to OTP `:logger` | Unresolved | OTP internals not audited (out of folder); no in-folder report covers them. No finding. |
| logger.ex | `:warn` backend level mapping lives in `Logger.Backends` (doc typo finding) | Confirmed | Real mapping `:warning→:warn` is in handler.ex:125-134; doc target `:warnings` is wrong. |

## Findings

#### `CSCAN-crossfile-formatter-format-event-nontotal`

- **Finding ID** — `CSCAN-crossfile-formatter-format-event-nontotal`
- **Severity** — Low
- **Confidence** — high
- **Files involved** — `lib-logger-backends-handler.ex.md`, `lib-logger-formatter.ex.md`, `THREAT_MODEL.md`
- **Issue** — `Logger.Formatter.format_event/2` is a documented public entry point of `Logger.Formatter` (THREAT_MODEL.md line 10). The formatter.ex report reviewed the file end-to-end and concluded "I found no functional defect reachable through the documented public API of this file," minting only prune/compile/systime findings; it never examined `format_message/3`'s invocation of the caller-supplied `report_cb` at `formatter.ex:291-302`. The handler.ex reviewer, reading across into `formatter.ex`, **verified by execution** that `format_message({:report, data}, %{report_cb: cb}, _)` calls `cb.(data)` — an arbitrary function that can raise any class or `throw` — and that a raising `report_cb` escapes `format_event/2` uncaught (`CSCAN-handler-format-event-uncaught-raise`). So the defect physically lives in `formatter.ex` but was missed by the per-file formatter review and only surfaced from the handler's cross-file read. The two reports contradict on whether `format_event/2` is total on documented input.
- **Resolution** — Upgrade the non-totality to a finding in the originating file (`formatter.ex`): `format_event/2`/`format_message/3` must not let an arbitrary `report_cb` (or a raising `Inspect`/`String.Chars` impl) escape its `IO.chardata()` contract. Confirm against `formatter.ex:283-333` source, then decide whether the guard belongs in the formatter (preferred, per handler.ex line 53) or as broadened containment in `handler.ex` log/2.

#### `CSCAN-crossfile-warn-keyerror-erl-level-invariant-unverified`

- **Finding ID** — `CSCAN-crossfile-warn-keyerror-erl-level-invariant-unverified`
- **Severity** — Low
- **Confidence** — medium
- **Files involved** — `lib-logger-backends-console.ex.md`, `lib-logger-formatter.ex.md`, `lib-logger-backends-handler.ex.md`
- **Issue** — Two files carry the same `Map.fetch!(colors, level)` sink and dismiss the same `KeyError` on the legacy `:warn` level by leaning on an invariant neither dismissal owns. `console.ex:252` (`color_event`) is ruled safe *only because* "`Logger.Backends.Handler.log/2` … always injects `[erl_level: erl_level | ...]` metadata (handler.ex:58)" that later overrides `level` to an OTP level; `formatter.ex` (`colorize/1`) rules out the twin KeyError because "a well-formed `:logger.log_event` carries OTP levels." Both dismissals rest on the claim that the level reaching `Map.fetch!` is always one of the eight OTP syslog keys. But the handler.ex report — the file both cite as the producer — never verifies that the `erl_level` injection at `handler.ex:58` is unconditional; its Ruled-out/Verified sections cover `gl` injection, `file` conversion, and level *mapping*, not `erl_level` *injection*. console.ex's own caveat (line 27) concedes "the safety of that line is an invariant of the upstream producers, not of this file." This is a systemic inherited risk: a single unverified producer invariant is the sole thing standing between two files and a `KeyError` crash.
- **Resolution** — Verify in `handler.ex` source that every event forwarded to the gen_event backends unconditionally carries `erl_level` with an OTP-level value (and that no documented path lets a caller suppress/overwrite it), then record that as an explicit verified invariant in handler.ex's report. If injection is conditional, both `console.ex:252` and `formatter.ex` `colorize/1` dismissals collapse into live defects and should be re-opened as defense-in-depth findings (e.g. `Map.get`/catch-all clause for the `:warn` key).
