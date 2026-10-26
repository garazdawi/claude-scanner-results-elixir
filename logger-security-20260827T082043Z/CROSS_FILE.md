# Cross-file findings

These per-file reports do not use the machine-readable `* Depends on:` marker; every cross-file dependency is asserted in prose. Notably, most were already *resolved in-report* — several files (`internal.ex`, `translator.ex`, `utils.ex`, `config.ex`) were explicitly re-reviewed against their neighbours and their hedges upgraded to verified statements. The ledger below enumerates every cross-file dependency assertion I found and resolves each against the cited neighbour's report.

## Asserted-dependency ledger

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| app.ex | formatter/utils/translator process the attacker-forwarded log content, not app.ex | Confirmed | All three reviewed as separate units; app.ex only touches operator config + internal `:logger` state. |
| app.ex | `:logger` app-env is operator/build-trusted (threat model) | Confirmed | THREAT_MODEL.md §Trust boundaries classifies operator config as trusted. |
| config.ex | `internal.ex:39` `Keyword.take(options,[:sync_threshold,:discard_threshold,:truncate,:utc_log])` confines the `put_env` keys | Confirmed | internal.ex report Phase-2 sink-1/sink-4 verified the allow-list bounds the downstream `put_env`. |
| config.ex | an out-of-tree direct caller of `Config.configure/1` would bypass the allow-list | Unresolved | No neighbour caller exists in-folder; documented `@moduledoc false` API-misuse caveat, not a defect. |
| console.ex | `Handler.log/2` always sets `erl_level` to a valid OTP level (rules out `Map.fetch!` KeyError) | Unresolved | handler.ex report does not state that `erl_level` is always populated; console verified it by its own end-to-end execution instead. |
| console.ex | message-content CR/LF/ANSI injection belongs to `Logger.Formatter`, not console | Confirmed | formatter.ex report acknowledges `prune`/`colorize` do not strip CR/LF (by-design, CWE-117). |
| console.ex | `retry_log` malformed-data path relies on `Logger.Formatter.prune/1` (out of file) | Confirmed | formatter.ex:358-365 is the code-point sanitiser, covered in formatter report. |
| console.ex | `configure/1` `put_env`/`compile` operate on operator/developer config | Confirmed | Consistent with internal.ex + threat model operator-trust. |
| handler.ex | formatting/inspection/truncation of hostile terms delegated to `Logger.Formatter` | Confirmed | formatter.ex + utils.ex reviewed; both cover truncation/inspection sinks. |
| handler.ex | `:logger` validates level against its fixed set before dispatch (rules out `FunctionClauseError`) | Unresolved | OTP `:logger` internals not in folder; asserted, not neighbour-verified. |
| internal.ex (sink 1/4) | `config.ex:34-40` `put_env` per option is the real sink; internal.ex only forwards | Confirmed | Read against config.ex; allow-list confinement confirmed both ways. |
| internal.ex (sink 1) | `console.ex:60` `handle_call({:configure,options})` is the arity-2 real sink | Confirmed | Read against console.ex report. |
| internal.ex (sinks 2-3) | `supervisor.ex:11-37` + `watcher.ex:15` install `backend` with no atom creation | Confirmed | supervisor.ex + watcher.ex reports corroborate module-atom pass-through. |
| internal.ex | `supervisor.ex:66` bounds restarts (`max_restarts:30, max_seconds:3`) | Confirmed | supervisor.ex report states the same intensity bound. |
| internal.ex | untrusted bytes forwarded as a backend module = caller-contract violation, out of folder | Unresolved | Arbitrary application callers not in folder callgraph; documented caveat. |
| supervisor.ex | crash-on-hostile-content logic lives in backend/handler/formatter, not supervisor | Confirmed | watcher.ex + console.ex report the actual crash paths; supervisor only picks restart intensity. |
| supervisor.ex | `backend`/`backends` originate from `Logger.add_backend/2` + `:logger` config (traced to logger.ex/internal.ex) | Confirmed | internal.ex + logger.ex reports corroborate provenance. |
| watcher.ex | the crashing legacy backend embeds attacker-logged data in its exception message | Unresolved | Backend-quality-dependent; the reviewer flags this as why the finding is Low/medium-conf (not a neighbour in folder). |
| watcher.ex | crash-loop can hit `Backends.Internal` `:rest_for_one` / supervisor `max_restarts` | Confirmed | supervisor.ex + internal.ex reports confirm the restart topology. |
| formatter.ex | `truncate_n`/`scan_inspect` live in `utils.ex` (out of file) | Confirmed | utils.ex reviewed; both functions covered and ruled safe. |
| formatter.ex | `Logger.Formatter.prune/1` is the downstream code-point sanitiser | Confirmed | Consistent across formatter/utils/console reports. |
| translator.ex | every clause runs under `Logger.Utils.translator/2` outer `try/rescue` (utils.ex:19-55) | Confirmed | utils.ex report verifies the rescue (34-41) + `else` mapping (42-54). |
| translator.ex | the `else`-branch (malformed translator return) escapes the rescue | Confirmed | utils.ex ruled-out entry documents the same `else`-outside-rescue gap. |
| translator.ex | `translator_inspect_opts` default `[]` bounds inspection cost | Confirmed | Verified by execution in translator report; consistent with formatter defaults. |
| utils.ex | `scan_inspect` has exactly one caller, `formatter.ex:310`, with default `%Inspect.Opts{}` (limit 200) | Confirmed | utils.ex read formatter.ex and verified sole-caller + default opts. |
| utils.ex | `truncate` source is formatter config (`options[:truncate] || fetch_env!`, default 8192) | Confirmed | Traced into formatter.ex:173 in utils report. |
| utils.ex | `~P`/`~W` template-position blowup requires attacker to control the format template (caller misuse) | Confirmed | Traced through the sole formatter caller; data-in-args path executed as bounded. |
| logger.ex | all dangerous transforms (format/truncate/translate/atom) happen in formatter/utils/translator | Confirmed | Those files reviewed; logger.ex proven to forward content verbatim by execution. |
| logger.ex | `flush/0` dispatch targets come from trusted `:logger.get_handler_config()` | Unresolved | OTP `:logger` handler registry out of folder; asserted trust-boundary rule-out. |
| mix.exs | no `mix.exs` disposition depends on sibling Logger source content | Confirmed | mix.exs report re-confirms it is a genuine scope boundary (inert literals only). |

## Findings

#### `CSCAN-crossfile-control-char-passthrough-deferral`

- **Finding ID:** `CSCAN-crossfile-control-char-passthrough-deferral`
- **Severity:** Low
- **Confidence:** medium
- **Files involved:** `lib-logger.ex.md`, `lib-logger-formatter.ex.md`, `lib-logger-backends-console.ex.md`, `lib-logger-backends-watcher.ex.md`, `lib-logger-translator.ex.md`
- **Issue:** This is a systemic responsibility-deferral pattern rather than a single-file contradiction. Attacker-influenced content carrying raw CR/LF/ANSI/CSI bytes flows through the whole stack un-escaped: `logger.ex` forwards message/metadata verbatim (verified, logger.ex:1050-1058) and explicitly says CR/LF handling "is the formatter's job"; `console.ex:122` says message-content injection "belongs to `Logger.Formatter`, not this file"; but `formatter.ex` (risks/caveats, `colorize` 243-277 / `prune` 358-365) states it deliberately does **not** strip CR/LF or CSI "by design" and did not raise it. So the layer every upstream file points at is the one that declines ownership. The net effect is confirmed reachable at two sinks that already carry their own per-file findings — `watcher.ex:60` (raw `Exception.format_exit` to stderr, `CSCAN-watcher-exit-reason-injection`) and `translator.ex:701/96/208` (raw `Exception.format_banner`/`Application.format_error`, the translator log-injection finding). The cross-file point is that the `watcher.ex` stderr path bypasses the formatter entirely, so the collective "the formatter owns sanitization" defense does not even apply there. Importantly, THREAT_MODEL.md §21-22 endorses the preserve-CR/LF policy as documented long-standing Logger behavior, which is why every reviewer correctly kept this Low — this finding does not overturn that, it records that the diffuse ownership is real and that no layer sanitizes.
- **Resolution:** Already substantially covered by the two per-file findings; no new source read is required. If a sanitizing decision is ever wanted it belongs at the formatter (`prune`/`colorize`) plus the `watcher.ex` stderr fallback (which has no formatter in its path), not at the individual deferring callers. No action needed to confirm — the pattern is verified across the cited reports.

## Note on Unresolved rows

The five Unresolved ledger rows (config.ex out-of-tree caller, console.ex `erl_level` invariant, handler.ex `:logger` level validation, internal.ex/watcher.ex backend-forwarding, logger.ex handler-registry trust) are coverage gaps where the neighbour report does not substantiate the asserted invariant — not defects. Per the reviewing protocol they remain ledger rows and are **not** promoted to findings; in each case the source file either verified the property by its own execution (console `erl_level`) or the invariant concerns code outside this folder's callgraph (OTP `:logger` internals, arbitrary application callers).
