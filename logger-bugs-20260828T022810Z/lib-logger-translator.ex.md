# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/translator.ex
<!-- sha256: 2d9c610af01347ff6f6a4a557aa2910d18c8ab8bdba6af35210dd28118f5c66e -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/translator.ex
<!-- sha256: 2d9c610af01347ff6f6a4a557aa2910d18c8ab8bdba6af35210dd28118f5c66e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (hedge-resolution pass over the first report)_

## Summary

`Logger.Translator.translate/4` is byte-identical to the released Elixir 1.21.0-dev source (prior `diff` → IDENTICAL against the `/opt/elixir` build), so behavioural tests against the installed `Logger.Translator` are authoritative for this source. In this pass I replaced the first report's one remaining hedge ("did not exhaustively enumerate every offender/child-debug keyword permutation") with executed enumeration, and I closed the reachability axis on the sole non-total helper (`crash_debug_key/1`) by reading OTP's `proc_lib` crash-report generator and then driving its exact key set through `translate/4`. I found no triggerable bug reachable through the documented contract.

## Bugfixes

No findings.

## Ruled out

- **`crash_debug_key/1` (line 609) is non-total and raises `CaseClauseError` on an unrecognized debug key.**
  *Confidence: high (dismissal).*
  Re-confirmed by execution that the helper raises: an `:error` `:crash_report` carrying `{:some_new_key, 123}` at `:debug` raised `CaseClauseError`, while the identical report at `:info` returned `{:ok, ...}` (`crash_info(_,_,_) -> []` short-circuits before the helper). *New evidence this pass:* I read OTP 29's `proc_lib.erl` and enumerated the exact key set a genuine crash report carries — `my_info_1/3` (lines 968–987): `pid, registered_name, process_label, error_info, ancestors, message_queue_len, messages, dictionary, trap_exit, status, heap_size, stack_size, reductions` (plus optional `initial_call`); and `make_neighbour_report/1` (lines 1073–1108): `pid, registered_name, process_label, initial_call, current_function, ancestors, message_queue_len, links, trap_exit, status, heap_size, stack_size, reductions, current_stacktrace`. Every one of those keys maps to a `crash_debug_key/1` clause or is specially handled (`current_stacktrace`, `initial_call`, `current_function`, `ancestors`, `process_label`). I then drove the *full* main-report + neighbour key set through `translate/4` at `:debug`, `:info`, and `:error` — all returned `{:ok, _, _}` (`elixir /tmp/full.exs`). The `CaseClauseError` is reachable only via a caller-fabricated report that violates the `:proc_lib` crash contract; identical to upstream.
  *Axes examined:* triggering-input (unknown key executed → raises), level-gating (`:debug` vs `:info`/`:error` executed), OTP-emitted-key-domain (read from `proc_lib` source **and** full set executed through `translate/4`), upstream divergence (prior `diff` IDENTICAL). *Not examined:* none of the axes tryable in this sandbox. (A hypothetical future OTP release adding a new crash-report key would surface the same gap in upstream Elixir too — a cross-project robustness item, not a divergence in this file.)

- **`format_reason/1` (line 658) misfiring its `{term, [_|_]}` "looks like `{exception, stacktrace}`" heuristic on a reason like `{:shutdown, [1,2,3]}`.**
  *Confidence: high (dismissal).*
  Re-verified: `format_stacktrace/1` raises internally on the non-stacktrace list, the surrounding `try/catch :error` (line 662) catches it, and the code degrades to `format_stop/1`, producing `{:ok, [... "\n** (stop) " | "shutdown: [1, 2, 3]" ...]}`. No crash, no wrong output.
  *Axes examined:* heuristic-false-positive input (executed), catch-recovery path (executed), upstream divergence (IDENTICAL). *Not examined:* none.

- **Clause overlap between `translate(_, _, :report, {:logger, %{label: label} = report})` (line 53) and `translate(_, :debug, :report, {:logger, [formatter_error: formatter] ++ data})` (line 231).**
  *Confidence: high (dismissal).*
  Disjoint by the second tuple element's type (map with `:label` vs. keyword list headed by `:formatter_error`); a `{:logger, map}` without a `:label` key falls through to the `:none` catch-all (line 247). Both the `formatter_error` path and every `%{label: ...}` path (`:gen_server`/`:gen_event`/`:gen_statem` terminate) were driven and returned correct results.
  *Axes examined:* pattern disjointness (type-level, both clauses can't match one term), each branch driven (executed), fall-through (executed). *Not examined:* none.

- **`report_crash/4` (line 527) calling `Keyword.get(dictionary, ...)` where `dictionary = crashed[:dictionary]` could be `nil`.**
  *Confidence: high (dismissal).*
  A `:proc_lib` crash report always carries a `:dictionary` entry — re-confirmed against `proc_lib.erl:981` (`{dictionary, cleaned_dict(Dict)}` is unconditionally in `my_info_1/3`). The full crash path was exercised end-to-end and produced correct output and metadata. Absence of `:dictionary` requires a contract-violating report (and `Keyword.get(nil, ...)` would `FunctionClauseError`, matching upstream).
  *Axes examined:* OTP always emits `:dictionary` (read from source), full path executed, upstream divergence (IDENTICAL). *Not examined:* none.

- **`report_gen_statem_terminate/2` (line 332) matching `reason: {kind, reason, stack}`.**
  *Confidence: high (dismissal).*
  OTP's `gen_statem` terminate report always supplies the 3-tuple `{Class, Reason, Stacktrace}`; the path was driven and returned correct chardata/metadata.
  *Axes examined:* 3-tuple contract, path executed. *Not examined:* none tryable here.

- **Deprecated `:format` paths (`** Generic server`, `Error in process`) at lines 154–202.**
  *Confidence: high (dismissal).*
  Both the debug-level Generic-server clause and the two `Error in process` clauses were driven and returned the documented `{:ok, chardata, metadata}` shape.
  *Axes examined:* each clause executed, min_level `:debug` branch executed. *Not examined:* none.

- **`report_supervisor`/`child_info`/`child_debug` clause coverage — the offender and child-debug keyword permutations the first pass did not enumerate.** *(Upgraded scope: first pass left this as a hedge; this pass enumerated it by execution.)*
  *Confidence: high (dismissal).*
  Driven exhaustively (`elixir /tmp/tperm.exs`): all three `report_supervisor` offender head clauses (`[{:pid,_},{:id,_}|_]`, `[{:nb_children,_},{:id,_}|_]`, `[{:pid,_}|_]`) × all four `sup_context/1` values (`:start_error`, `:child_terminated`, `:shutdown`, `:shutdown_error`), plus the `offender_reason({:EXIT,_}, :start_error)` branch — every combination returned `{:ok, ...}`. Every `child_debug_key/2` clause (`:restart_type`, `:shutdown`, `:child_type`, `:significant` true and false) plus its catch-all (`_,_ -> []`, driven with `:some_unknown_key`) returned `{:ok, ...}`. `child_debug_key/2` is total (has a catch-all at line 508), unlike `crash_debug_key/1`.
  *Axes examined:* every offender clause × every context (executed), every child_debug key incl. unknown (executed). *Not examined:* none.

## Risks / caveats — Did I verify this?

- **Byte-identity:** Verified in the first pass (`diff` against `/opt/elixir` → IDENTICAL). Behavioural results from the installed `Logger.Translator` are authoritative for this exact source. Toolchain this pass: Erlang/OTP 29 (erts-17.0.2, JIT), Elixir 1.21.0-dev.
- **Offender/child-debug permutations:** The first pass's "did not exhaustively" hedge is now discharged — every offender clause, every supervisor context, and every child-debug key (including the catch-all) were executed and all returned `{:ok, ...}`. No permutation was left asserted.
- **`crash_debug_key/1` reachability:** Closed by executed evidence, not invariant reasoning — I read the OTP key domain from `proc_lib.erl` *and* pushed that full domain through `translate/4` at all three levels. The only input that raises is a key OTP never emits.
- **Contract boundary:** Every dismissal above rests on the documented contract that `message` is a conforming `:logger.report()`. A caller that forwards non-OTP report data through `translate/4` could still hit `crash_debug_key/1` (`CaseClauseError`) — a latent robustness gap that is identical to upstream and out of scope for a contract-conforming bug.

## Dropped in re-review

None. No prior finding was dropped (the first pass had no findings). No dismissal was upgraded to a finding: the one dismissal whose reasoning the first pass had not fully executed (`crash_debug_key/1` reachability) is now supported by executed evidence across every axis tryable in this sandbox, so it correctly remains ruled out rather than being downgraded to a low-confidence finding.
