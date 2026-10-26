# Cross-file findings

## Asserted-dependency ledger

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| event_manager.ex | `runner.ex:43,76` — `handler`/`opts` are trusted config (`ExUnit.RunnerStats` literal + `opts[:formatters]`) | Confirmed | runner.ex report confirms module-queue/opts are developer suite config; event_manager reviewer read runner.ex directly. |
| callbacks.ex | `on_exit_handler.ex:21–33,139–145` — `name_or_ref` used only as keystore key, `callback` run as guarded 0-arity closure | Confirmed | on_exit_handler.ex Ruled-out entries confirm keystore-only key and guarded `exec_callback/1` closure. |
| callbacks.ex | `on_exit_handler.ex`/`server.ex` — `ExUnit.Server` ETS table ownership | Confirmed | server.ex + on_exit_handler.ex both confirm `:public` table created in `ExUnit.Server.init/1`. |
| capture_io.ex | `capture_server.ex` — named-device hijack (`Process.register`) is caller-chosen/documented | Confirmed | capture_server.ex traces both callers, rules device/formatter dispatch trusted. |
| capture_log.ex | `capture_server.ex` — trust handling of forwarded `:formatter {mod,cfg}` opts | Confirmed | capture_server.ex confirms `formatter_mod.format/2` dispatch is developer-chosen, co-tenant-only. |
| case_template.ex | `case.ex:368` — `__keys__/1` is allow-list `Keyword.take`, no atom/dispatch sink | Confirmed | case_template reviewer read case.ex:368 directly; consistent with case.ex report. |
| cli_formatter.ex | `formatter.ex` — inspect/format crashes on pathological terms belong to formatter | Confirmed | formatter.ex report reproduces the crash (its standing Low finding), corroborating the deferral. |
| diff.ex | `formatter.ex:558–565` — sole caller wraps `Diff.compute` in `Task.yield(1500)`/`shutdown` (DoS bound) | Confirmed | formatter.ex confirms the 1500 ms Task wrapper is the only call site. |
| runner.ex | `server.ex:16–37,140–173` — module queue sole ingress `add_module/2`, trusted atoms | Confirmed | server.ex report confirms `add_module/2` carries compile-time atoms + developer config only. |
| runner.ex | `case.ex:285–310` — `:tmp_dir` tag is developer-set string, "removed before created" documented | Confirmed | case.ex report confirms compile-time developer-controlled tag. |
| runner_stats.ex | `failures_manifest.ex:70–74` — real deserialization sink lives there | Confirmed | failures_manifest.ex has the verified Medium `binary_to_term`-without-`:safe` finding. |
| runner_stats.ex | `ex_unit.ex:351` — `:failures_manifest_path` is documented operator/CI config | Confirmed | ex_unit.ex report confirms `configure/1`/`start/1` option surface. |
| filters.ex | `failures_manifest.ex:70–74` — `failure_info/1`/`fail_all!/1` forward to the unsafe decode sink | Confirmed | filters.ex reviewer read failures_manifest.ex; matches its finding. |
| filters.ex | `runner.ex:287–311` — `eval/4` receives full `tests` collection; regex from `opts[:include/exclude]` | Confirmed | runner.ex report confirms full-collection pass + developer-derived filters. |
| case.ex | `doc_test.ex` / `register_test/6` callers forward only developer/dependency names | Confirmed | doc_test.ex report confirms doc-driven names are trusted/self-inflicted; sink length-bounded regardless. |
| supervisor.ex | `ex_unit.ex:549` — sole caller `fetch_test_supervisor/0` passes literal `[]` | Confirmed | supervisor.ex reviewer read ex_unit.ex:549 directly (test-supervisor accessor in ex_unit.ex report). |
| server.ex | `on_exit_handler.ex` — ETS read/write/delete integrity (can a planted entry crash the reader?) | Confirmed | on_exit_handler.ex report addresses the ETS contract: all writers trusted pids, poisoning is co-tenant/out-of-scope. |
| filters.ex | `mix/tasks/test.ex` — callers of `parse/1`/`parse_paths/1` feed only developer-CLI strings | **Unresolved** | Caller module absent from this checkout; filters.ex explicitly could not verify; already carried as its standing Low finding's caveat. |
| doc_test.ex | `Code` module — `Code.fetch_docs` internal `binary_to_term` safe/unsafe | **Unresolved** | `Code` module is outside this folder; no report exists; scope-boundary, not a defect claim. |
| assertions.ex | (unnamed) caller that might route exception messages into a shell/query/EEx | **Unresolved** | No concrete second file named; generic hedge, not a resolvable dependency. |

## Findings

No cross-file findings.

Every concrete cross-file dependency in this report set was resolved in the re-review pass by the per-file reviewer reading the cited neighbour directly, and each is corroborated by that neighbour's own report — no file asserts another is safe where the other's report says otherwise. The two genuinely load-bearing security sinks (the unsafe `failures_manifest.ex` `binary_to_term` reached via `runner_stats.ex`/`filters.ex`, and the `diff.ex`↔`formatter.ex` `Diff.compute` blow-up/crash) are already reported as verified findings *in the files that own the sink*, with their cross-file reachability chains consistent across both endpoints — so there is nothing a per-file view missed, only findings correctly localized.

The three **Unresolved** rows are coverage gaps, not defects: `mix/tasks/test.ex` and the `Code` module lie outside this checkout, and the assertions.ex hedge names no concrete second file. Per the ledger discipline they remain rows recording what was not checked, and none is promoted to a finding.
