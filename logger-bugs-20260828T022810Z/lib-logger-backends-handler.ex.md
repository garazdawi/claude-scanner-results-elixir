# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/handler.ex
<!-- sha256: ec8361a9c2d59971eab4c11da898175437f75adce9394814fce3ff24b6320a5e -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/handler.ex
<!-- sha256: ec8361a9c2d59971eab4c11da898175437f75adce9394814fce3ff24b6320a5e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T02:46:00Z (first pass 2026-08-28T02:39:12Z)_

## Summary

Two issues survive the re-review. The primary defect is unchanged: `changing_config/3` is non-total and crashes with `FunctionClauseError` on any non-`:refresh` OTP config operation. The re-review then resolved the first pass's hedge over `Logger.Formatter.format_event/2` by reading `formatter.ex` **and executing the path**: `format_event/2` is *not* total — a caller-supplied `report_cb` (or any term whose `Inspect`/`String.Chars` impl raises) makes it raise a non-`ArgumentError` exception or `throw`, both of which propagate straight past this file's `rescue ArgumentError` / `catch :exit` and crash the handler callback. That is now a low-confidence finding rather than a hedged out-of-scope note. The `system_time_to_date_time_ms/2` hedge was cleared as benign for the inputs the handler passes.

## Bugfixes

#### `CSCAN-handler-changing-config-nontotal`
- **Finding ID:** `CSCAN-handler-changing-config-nontotal`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `elixir/lib/logger/lib/logger/backends/handler.ex:19` — `changing_config/3`
- **Class:** Missing clause / non-total function (also a behaviour-contract violation)
- **Issue:** The `:logger_handler` behaviour specifies that `changing_config/3` is invoked for *every* config change on the handler, with the operation being `:set` or `:update` and the third argument being the full new handler-config map. This module implements only the single clause `changing_config(:update, config, %{config: :refresh})`. Consequently any config change to the registered `Logger` handler other than the internal refresh — e.g. the documented OTP calls `:logger.set_handler_config(Logger, :level, :info)` or `:logger.update_handler_config(Logger, :filters, [])` — routes into this callback with an operation/shape that matches no clause and raises `FunctionClauseError`. OTP wraps it and the whole config operation fails with `{:error, {:callback_crashed, {:error, :function_clause, ...}}}`, leaving the change unapplied. The supported `:refresh` path (used by `Logger.Backends.Internal`, `internal.ex:40`) works, so only non-refresh reconfiguration is affected.
- **Suggested fix:** Add a catch-all clause that accepts the other operations and returns the new config unchanged, e.g. `def changing_config(_op, _old, new), do: {:ok, new}` (placed after the `:refresh` clause). If the intent is to reject foreign updates, return an explicit `{:error, reason}` instead of letting it raise.
- **Impact / threat:** A caller who manages OTP log handlers uniformly and adjusts the level or filters of the `Logger` bridge handler (a legitimately registered, introspectable handler) gets an opaque `callback_crashed` failure and the change silently does not take effect. No untrusted-data path is involved; the trigger is a documented OTP reconfiguration API against a documented behaviour callback.
- **Evidence:** `verified`.
  Command: `elixir` (built from source under review) running
  ```elixir
  Application.ensure_all_started(:logger)
  {:ok, _} = Logger.Backends.Internal.start_link([])
  :logger.update_handler_config(Logger, :config, :refresh)   #=> :ok
  :logger.set_handler_config(Logger, :level, :info)
  :logger.update_handler_config(Logger, :filters, [])
  ```
  Output:
  ```
  refresh: :ok
  set level: {:error, {:callback_crashed, {:error, :function_clause,
     [{Logger.Backends.Handler, :changing_config, [:set, %{...}, %{...}], [file: ~c"lib/logger/backends/handler.ex", line: 19]} ...]}}}
  update filters: {:error, {:callback_crashed, ...}}
  ```

#### `CSCAN-handler-format-event-uncaught-raise`  *(Upgraded from hedged assumption in first pass)*
- **Finding ID:** `CSCAN-handler-format-event-uncaught-raise`
- **Severity:** Low
- **Confidence:** low
- **Location:** `elixir/lib/logger/lib/logger/backends/handler.ex:49` (`Logger.Formatter.format_event(event, truncate)`), with the too-narrow guard at lines 62–65 (`rescue ArgumentError` / `catch :exit, reason`).
- **Class:** Crash-on-untrusted-input / incomplete error containment (Phase 1 sink class: untrusted-data-triggered crash).
- **Issue:** `log/2` calls `Logger.Formatter.format_event/2` inside a body guarded only by `rescue ArgumentError -> {:error, :noproc}` and `catch :exit, reason -> {:error, reason}`. Having now read `formatter.ex`, `format_event/2 → format_message/3` is **not total in `ArgumentError`/`:exit`**:
  - `format_message({:report, data}, %{report_cb: callback}, truncate)` (`formatter.ex:291–302`) invokes an **arbitrary caller-supplied function** `callback.(data)` (report_cb is `meta`-controlled per the folder threat model). That function can raise any exception class or `throw`.
  - `format_report/2` (`formatter.ex:315–321`) calls `Kernel.inspect/2`, which raises whatever a term's custom `Inspect` implementation raises (commonly not `ArgumentError`).
  A raise other than `ArgumentError`, or a `throw`, is caught by neither the `rescue ArgumentError` nor the `catch :exit` clause, so it propagates out of `log/2` and crashes the handler callback. OTP's `:logger` core contains the crash (reports it and can disable the `Logger` bridge handler after repeated failures), so this is degradation, not a VM crash — hence Low severity — but it drops the event and is reachable from a documented OTP logging call.
- **Suggested fix:** Either broaden the containment in `log/2` (e.g. add `rescue e -> {:error, ...}` / `catch :throw, _`), or push the responsibility into the formatter so `format_event/2` never escapes its documented `IO.chardata()` contract. Note the obligation to be total properly belongs to `Logger.Formatter`; this file's contribution is that its `rescue ArgumentError` advertises containment it does not actually provide.
- **Impact / threat:** A caller logging a `{:report, _}` event with a `report_cb/1` that raises (or logging a struct whose `Inspect` impl raises) triggers a handler-callback crash and event loss; repeated occurrences can lead OTP to remove the Logger bridge handler.
- **Evidence:** `verified` (executed).
  Command: `elixir` (built from source under review) running `Logger.Formatter.format_event/2` and `Logger.Backends.Handler.log/2` directly:
  ```elixir
  ev = %{level: :error, msg: {:report, %{foo: 1}},
         meta: %{gl: self(), report_cb: fn _ -> raise RuntimeError, "boom" end}}
  cfg = %{config: %{counter: :counters.new(1,[:atomics]), truncate: 8096,
                    utc_log: false, thresholds: {20, 500}}}
  Logger.Backends.Handler.log(ev, cfg)
  ```
  Output:
  ```
  report_cb raise:  {:raised, RuntimeError}          # format_event/2 alone
  report_cb throw:  {:caught, :throw, :nope}          # throw variant
  handler.log report_cb raise: {:handler_raised, RuntimeError, "boom"}   # propagates past rescue ArgumentError
  ```
  The `{:handler_raised, RuntimeError, ...}` line is the load-bearing observation: the exception escaped `log/2`.

### Ruled out
- **`%{gl: gl} = metadata` in `log/2` (line 46) raising on missing `:gl`** — Ruled out. Note the raise would be a `MatchError` (executed: `Logger.Backends.Handler.log(%{level: :info, msg: {:string,"hi"}, meta: %{time: 0}}, cfg)` → `{:handler_raised, MatchError}`), which is *not* caught by `rescue ArgumentError` — so the dismissal rests entirely on gl always being present. Verified by execution that OTP injects it: a primary filter installed via `:logger.add_primary_filter/2` observed `gl` present for `:logger.log(:error, "m", %{})` (empty metadata), the `Logger.error` macro, and a `{:report, _}` with `report_cb`, all reporting `{true, [:pid, :time, :gl | ...]}`. Primary filters run in `do_log` after `add_default_metadata` and before any handler, so a handler reached via the documented pipeline always sees `gl`.
  - **Axes examined:** msg-shape (`{:string,_}`, macro/format, `{:report,_}`), empty-vs-populated caller metadata, filter-vs-handler ordering. **Not examined:** none reachable via the documented `:logger.log` pipeline (direct `:gen_event`/handler invocation bypassing `do_log` is out of scope).
- **`List.to_string(file)` in `erlang_metadata_to_elixir_metadata/1` (line 112) raising when `file` metadata is a binary rather than a charlist** — The whole function body (lines 101–116) is enclosed by `rescue _ -> Map.to_list(metadata)` at lines 117–118, so a raising `List.to_string/1` degrades to the original metadata list. The `rescue _` here catches *any* class (unlike the narrow one in `log/2`), so this is genuinely contained. Verified by inspection of the `rescue` scope.
- **`notify(:sync, msg)` calling `:gen_event.sync_notify(nil, msg)` when `Logger` is unregistered (lines 68–74)** — `pid == self()` is false for `nil`, so it calls `sync_notify(nil, …)`, producing an `:exit`/`:noproc` caught by `catch :exit, reason -> {:error, reason}` at lines 64–65. Handled, not a crash. (Distinct from the `format_event` finding: `:exit` *is* in this file's catch set; foreign raises/throws are not.)
- **`threshold/1` never decrementing the counter on `:discard` (lines 82–96)** — Counter reconciliation against `message_queue_len` is done deliberately in `Logger.Backends.Config.update_counter/2` (verified in `config.ex:62–96`: reads `total`, reads `:message_queue_len`, then `:counters.add(counter, @counter_pos, length - total)`), the documented, intentionally-lossy overload-protection design shared via the same `@counter_pos 1` atomic. Not a defect in this file.
- **`erlang_level_to_elixir_level/1` collapsing `:warning→:warn` and `:notice→:info` (lines 125–134)** — Intentional legacy-backend level mapping (the gen_event backends use the old `:warn` name); total over the eight real OTP event levels plus the defensive `:all`/`:none`. Not a bug.
- **`adding_handler/1` (defaults win under user) vs `changing_config` refresh (defaults win over existing) merge-direction asymmetry (lines 16, 20)** — Deliberate: `:refresh` re-pulls fresh `Application` env values and overwrites live config, while `adding_handler` lets initial user config override defaults. The internal `counter` key is preserved in both because `default_config/0` (lines 27–36) never contains it. Consistent with intent.

## Risks / caveats
- The `changing_config/3` finding's reachability depends on a caller targeting the `Logger` bridge handler with a non-`:refresh` OTP config operation; Elixir's own Logger app only ever calls it with `:refresh` (verified `:ok`), so normal Logger usage does not hit it. It is nonetheless a genuine violation of the documented `:logger_handler` `changing_config/3` contract.
- The `format_event` finding is Low/low because OTP's `:logger` core wraps handler-callback invocations, so the propagated exception is reported and contained rather than crashing the VM; the concrete harm is event loss and possible handler removal under repeated failures. The trigger (a raising `report_cb` or a raising `Inspect` impl) is caller-controllable but not something normal well-formed logging produces.
- Verification ran against the `elixir`/`iex` toolchain in the sandbox (built from the source under review). For the `changing_config` case I started `Logger.Backends.Internal` explicitly to register the handler.

## Did I verify this?
- **`Logger.Formatter.format_event/2` totality (prior hedge)** — **Verified by execution, not assumed.** Read `formatter.ex:283–333` and ran the path: a `report_cb/1` raising `RuntimeError` and a `report_cb/1` doing `throw` both escape `format_event/2`, and the `RuntimeError` was observed escaping `Logger.Backends.Handler.log/2` past its `rescue ArgumentError`. Upgraded to finding `CSCAN-handler-format-event-uncaught-raise`.
- **`system_time_to_date_time_ms/2` (line 57) raising (prior hedge)** — **Cleared as benign.** Read `formatter.ex:394–405`. The handler passes only a guarded non-negative integer (`is_integer(time) and time >= 0`, else `:os.system_time(:microsecond)` at lines 51–55); `rem/2`, `div/2`, and `:calendar.system_time_to_{local,universal}_time/2` do not raise for such input. No finding.
- **`gl` always present in the documented pipeline** — **Verified by execution** via a primary filter (see Ruled out); not assumed from OTP docs this time.
- **Counter reconciliation lives in `Config`** — **Verified by reading** `config.ex:62–96`.

## Dropped in re-review
None. Both prior findings are retained (the `changing_config` finding unchanged; the former out-of-scope hedge on `format_event` is promoted to an explicit low-confidence finding rather than dropped). All six prior ruled-out entries are retained, two of them (gl-injection, counter-reconciliation) strengthened with newly executed / cross-file evidence.
