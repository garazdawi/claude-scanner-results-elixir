# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/handler.ex
<!-- sha256: ec8361a9c2d59971eab4c11da898175437f75adce9394814fce3ff24b6320a5e -->

_Analysed on 2026-08-27T08:31:46Z_

Confirmed: **0 atoms** created over 100k conversions of distinct external byte sequences, and the metadata helper's `rescue _` contains every exception class those conversions can throw (`ArgumentError`, `FunctionClauseError`, `UnicodeConversionError`). The file's conversions are atom→string / int→string / charlist→binary — none go the dangerous direction (string→atom).

I have enough to conclude. Report follows.

---

## Summary

`Logger.Backends.Handler` is orchestration glue for the legacy `:gen_event` backend path — it thresholds, timestamps, reshapes metadata, and forwards events to the trusted `Logger` process — and contains no sink that turns attacker-influenceable log content into code, atoms, paths, files, network calls, or unbounded work; no security findings.

## Security

`No findings`

The event `message`/`meta` reaching `log/2` can carry data the surrounding application forwarded from the network (per the folder threat model), so the trust boundary is real. But every sink in this file is benign with respect to that data:

- **Formatting/inspection/truncation of hostile terms** (the actual risk surface) is delegated to `Logger.Formatter.format_event` and `system_time_to_date_time_ms` — *other files*, out of scope here. This file only passes the event through.
- **Metadata reshaping** (`erlang_metadata_to_elixir_metadata/1`, `form_fa/2`) touches attacker-influenceable values (`meta.file`, `meta.mfa`) but only via `Atom.to_string` / `Integer.to_string` / `List.to_string` — conversions that go value→string, never string→atom. Verified: 0 atoms created over 100k conversions of distinct external byte sequences (`atom delta over 100k conversions: 0`). Every exception these can raise (`ArgumentError`, `FunctionClauseError`, `UnicodeConversionError` — all observed empirically) is contained by the helper's `rescue _`.
- **`threshold/1`** writes only to a shared `:counters` reference (bounded 64-bit) and reads it back; no external value indexes or sizes anything.
- **`notify/2`, `filesync/1`** are `:gen_event` message passes to the fixed, trusted `Logger` process — no derived module/name dispatch, no `apply/3`.
- **Config functions** (`adding_handler`, `changing_config`, `filter_config`, `default_config`) operate on operator-trusted application env with pure `Map` ops.

No `to_atom`/`Module.concat`, no `eval`/`apply` on derived references, no `binary_to_term`, no `File`/`Path`, no OS/shell, no network, no regex-from-input.

## Risks / caveats

- The genuinely interesting attack surface for hostile log content — truncation byte-math, UTF-8 boundary handling, `~p`/`~w` inspection cost, `String.to_unsafe_atom` in pattern compilation — lives in `Logger.Formatter` and `Logger.Utils`, which I did not review here. My "no findings" verdict is scoped strictly to `handler.ex`; it is not a clearance of the formatting path it calls into.
- I verified conversion/atom/rescue behavior by direct reproduction. I did not stand up a full `:logger` handler and drive an end-to-end log event through the running module (the private functions aren't callable in isolation); the private-function logic was reproduced faithfully rather than exercised in situ. I judged this sufficient because the functions are small and their sinks are unambiguous.

### Ruled out

- **`erlang_level_to_elixir_level/1` has no catch-all clause** — an out-of-set level atom raises `FunctionClauseError`, which the `log/2` `rescue ArgumentError` does *not* catch, so it would propagate and (after repeated failures) cause OTP `:logger` to disable the handler. Not reachable via a documented path: `:logger` validates the level against its fixed set (`emergency…debug`, plus `none`/`all`) before dispatching to handler callbacks, and the clause list is exhaustive over exactly that set. Latent robustness gap, not a security issue.
- **`threshold/1` read-modify-check is non-atomic** — `:counters.add` then a separate `:counters.get` lets a concurrent logger interleave, so `value` may reflect other processes' increments. This is an approximate overload gauge by design; a small skew only nudges the async/sync/discard decision. No integrity or safety consequence.
- **Attacker-forwarded metadata values are not truncated in this file** (`file` charlist → `List.to_string`, `mfa` → `form_fa`) — size is bounded by whatever the application itself chose to place in metadata, and per-field truncation for rendering is the formatter's responsibility. No amplification introduced here.
- **`notify(:sync, msg)` with `Logger` unregistered** → `:gen_event.sync_notify(nil, …)` raises/exits, but is caught by `log/2`'s `rescue ArgumentError` / `catch :exit → {:error, reason}`. Robustness, contained.
- **`log/2` rescues only `ArgumentError`** — a non-`ArgumentError` from `format_event` would propagate. That is a `Logger.Formatter` concern plus standard OTP handler-removal behavior, not a sink in this file.

