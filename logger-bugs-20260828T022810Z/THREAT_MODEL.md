I have enough context to write the summary.

# Correctness/Bug Profile — `elixir/lib/logger` (Elixir Logger)

## Protocol / standard implemented
This is Elixir's **Logger** application: an Elixir-side integration on top of Erlang/OTP's `:logger` (Erlang's `kernel` logging framework, governed by the OTP `logger` design — `logger_chapter`, `logger_handler`, `logger_filters` behaviours — not an IETF RFC). It provides the primary handler, a formatter, a translator for OTP crash/supervisor reports, an overload-protection counter, and a legacy `:gen_event`-based backend system. It is not a network protocol; "untrusted input" is log-event data and format strings, not packets.

Documented public entry points:
- `Logger` (in `logger.ex`) — `log/2-3`, level macros, `metadata/0-1`, `configure/1`, `add_backend/2`, `add_translator/1`, etc.
- `Logger.Formatter` — `new/1`, `compile/1`, `format/5`, `format_event/2`, `truncate/2`, `prune/1`, `system_time_to_date_time_ms/2`. This is the caller-facing custom-formatter toolkit.
- `Logger.Translator.translate/4` — the `c:translate/4` callback implementation.
- `Logger.Backends.Internal` — `add/2`, `remove/2`, `configure/1-2` (legacy backend API).
- `Logger.Utils` — `@moduledoc false`, but its functions are `:logger` filter callbacks (`translator/2`, `process_level/2`, `silence_logger_exit/2`) and `scan_inspect/4`, `truncate_n/2`.

## Trust / contract boundaries
- **Log-event data** crosses into `Logger.Backends.Handler.log/2`, `Formatter.format/2`, and `Utils.translator/2` as an Erlang `:logger.log_event()` map. `msg`, `meta`, `level` are effectively caller/peer-controlled. Fields like `meta.time`, `meta.gl`, `meta.report_cb`, `meta.mfa`, `meta.file` are guarded but arbitrary: e.g. `report_cb` is an *arbitrary function* invoked in `format_message/3` (`callback.(data)`), and `meta.file` is `List.to_string`'d.
- **Format strings** (`Formatter.compile/1`, `Utils.scan_inspect/4`) are operator-supplied config but flow into `:io_lib.scan_format`. Note `compile/1` uses `String.to_unsafe_atom` on the pattern code (atom-table growth if operator-controlled patterns are dynamic).
- **Metadata values** reach `Formatter.metadata/2`, which enumerates many type clauses (pid/ref/port/atom/list/String.Chars) — the contract is "printable or dropped", and the catch-all silently drops.
- **Config** (`sync_threshold`, `discard_threshold`, `truncate`, `utc_log`) comes from `Application` env via `fetch_env!` — a missing key crashes at handler-config time.

## Failure-mode model
- `Handler.log/2` wraps in `rescue ArgumentError -> {:error, :noproc}` / `catch :exit`; anything else propagates. The translator path in `Utils.translator/2` has a broad `rescue` that converts failures into a `{:string, chardata}` event — so a raising translator degrades to a formatted-error message rather than a crash, but a translator returning an *unexpected tuple shape* falls through the `else` clauses and would `case_clause`.
- `Formatter` is dense with clause-based dispatch (`output/5`, `metadata/2`, `add_reset/1`, `prune/1`, `format_message/3`) — classic **crash-on-stray-input** and **silent-drop** territory. `metadata(_, list) -> nil` silently discards non-charlist lists; a wrong-arity guard would drop valid metadata.
- `truncate_n/2` / `fix_binary/2` in `Utils` do hand-rolled UTF-8 byte accounting (the 13-byte look-back, code-point width tables) — off-by-one truncation or malformed-binary edge cases are the likely bug locus, and it can raise `ArgumentError` (caught in `wrapped_truncate` for lists only, **not** the binary branch).
- Counter arithmetic in `Config.update_counter/3` and `Handler.threshold/1` (`:counters.add(counter, 1, length - total)`) is documented as intentionally lossy — reviewers should not "fix" the drift comments as bugs.

## Common bug patterns for this family (stdlib/logging)
- Guard/clause typos (`andalso`/`orelse`, `in 0..127` boundary off-by-one in `truncate_n` width tables).
- Accumulator dropped on the empty/edge clause of the recursive `metadata/1`, `add_reset/1`, `truncate_n_list/3`, `prune/1` walkers.
- Level-mapping table errors in `erlang_level_to_elixir_level/1` (note `:warning -> :warn`, `:notice -> :info` collapses — intentional legacy mapping, not a bug).
- Returning a tuple shape the `translate/4` contract forbids (must be `{:ok,_}|{:ok,_,_}|:skip|:none`); `Utils.translator/2`'s `else` only handles those four.

## Supervisor / process structure
- `Logger.Backends.Internal` (`Supervisor`, `:rest_for_one`) owns: the `:gen_event` manager registered as `Logger`, a `Watcher` for `Backends.Config`, and `Backends.Supervisor`. Because it's `rest_for_one`, if `:gen_event` dies everything after restarts.
- `Logger.Backends.Supervisor` (`:one_for_one`, `max_restarts: 30, max_seconds: 3`) supervises one `Watcher` per backend.
- `Logger.Backends.Watcher` (`GenServer`, traps exits) wraps each `:gen_event` handler; a crashing handler surfaces as `{:gen_event_EXIT, ...}`. Note its `init/1` **recurses** (`init({handler,args})`) after deleting an existing handler — an infinite-loop risk if delete keeps returning non-error.
- `Logger.Backends.Config` is a `:gen_event` handler doing periodic self-scheduled counter checks (`Process.send_after`).

## Cross-file invariants
- `Handler.log/2` and `Formatter.format/2` both re-derive `system_time`/`date_time_ms` with the *same* `meta.time` guard — they must stay in sync; a divergence silently changes timestamps.
- `Formatter.truncate/2` delegates the actual byte-walk to `Utils.truncate_n/2`; the `n >= 0` post-check in `truncate/2` assumes `truncate_n` returns a *negative* remainder exactly when it overran — the "(truncated)" marker correctness depends on that contract.
- `Utils.translator/2` (the filter) assumes `Translator.translate/4` honours the four-value return contract; a new report clause in `translator.ex` returning something else breaks the filter's `else`.
- `Backends.Config` being the *sole* remaining handler is the sentinel `Watcher.logger_has_backends?/0` uses to decide whether to print to stderr — adding another always-present handler would silently suppress that fallback.
- `Handler` and `Config` share the same `@counter_pos 1` atomics counter created in `Internal.init/1`; the handler increments, the config process reconciles against `message_queue_len`. Overload protection correctness spans all three files.