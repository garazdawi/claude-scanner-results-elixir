# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar.ex
<!-- sha256: 1d09a24467a01bd10b0f545ade312c69afc7dfee85af259e0dd221785401701f -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar.ex
<!-- sha256: 1d09a24467a01bd10b0f545ade312c69afc7dfee85af259e0dd221785401701f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28. Hedge phrases from the first pass ("outside this file") resolved by reading callgraph neighbours (`calendar/date.ex`) and by pushing values through the live `Elixir 1.21.0-dev` toolchain at `/opt/elixir/bin/elixir`._

## 1. Summary
The `Calendar` module is mostly type/behaviour declarations plus small pure helpers. The meaningful attack surface is the `strftime/3` format-string engine: a super-linear (cubic) resource-amplification path and one unguarded `%f` crash, both reproduced by execution. In addition, this re-pass upgrades one previously-hedged dismissal — caller-map-controlled calendar dispatch — to a low-confidence finding after executing a proof that the dispatch actually fires on an attacker-chosen module.

## 2. Security

#### `CSCAN-calendar-strftime-preferred-amplification`
- **Finding ID:** `CSCAN-calendar-strftime-preferred-amplification`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `calendar.ex:773-780` (`format_modifiers "c"`), `calendar.ex:892-899` (`"x"`), `calendar.ex:914-921` (`"X"`); incomplete guard at `calendar.ex:676`; entry `strftime/3` at `calendar.ex:641`.
- **Class:** Resource consumption
- **Issue:** `%c`/`%x`/`%X` recursively re-`parse/4` the `preferred_datetime`/`preferred_date`/`preferred_time` option strings (`calendar.ex:776`, `895`, `917`). The `preferred_*_invoked` flags cap *recursion depth* at 3, but nothing caps the *number* of directives in a string or the *total* output size. Each level multiplies, so work grows as (#`%c`)×(#`%x`)×(#`%X`) — cubic in input. The `@strftime_max_width` (1024) check at `calendar.ex:676` bounds only one directive's padding width, not directive count or multiplicative expansion.
- **Impact / threat:** `strftime/3` is documented public API; the `## User Options` section documents `preferred_datetime`/`preferred_date`/`preferred_time` as user options, so a multi-tenant/localization app storing per-user preferred formats lets an attacker supply all four inputs and expand ~400 bytes to 8 MB in ~1 s on one call — enough to OOM/wedge the BEAM (DoS). Even format-string-only (default options) gives ~170× linear amplification via width padding.
- **Evidence:** `verified` (re-executed this pass). `Calendar.strftime(~U[2019-08-26 13:52:06.048531Z], String.duplicate("%c",200), preferred_datetime: String.duplicate("%x",200), preferred_date: String.duplicate("%X",200), preferred_time: "Z")` → `in=400 out=8000000 ms=918.6`. Depth bound confirmed polynomial-not-infinite: `preferred_datetime: "%c"` → `RAISED tried to format preferred_datetime within another preferred_datetime format`. Per-directive width cap works: `"%99999999Y"` → `RAISED invalid strftime format: width must be at most 1024`.
- **Suggested fix:** Thread a running output-byte counter through `parse/4`/`format_modifiers/6` and raise `ArgumentError` once cumulative output exceeds a bound (e.g. a multiple of `@strftime_max_width` or input length), and/or cap total directives processed.

#### `CSCAN-calendar-strftime-f-precision-crash`
- **Finding ID:** `CSCAN-calendar-strftime-f-precision-crash`
- **Severity:** Low
- **Confidence:** high
- **Location:** `calendar.ex:789-799`, specifically `binary_part(0, max(precision, 1))` at `calendar.ex:796`.
- **Class:** Resource consumption (availability / uncaught crash)
- **Issue:** `%f` computes `microsecond |> Integer.to_string() |> String.pad_leading(6, "0") |> binary_part(0, max(precision, 1))`. `String.pad_leading/3` only *adds* padding, so the intermediate string is `max(6, digits(value))` bytes. If `precision` exceeds that length, `binary_part/3`'s length argument is out of range and raises. No guard clamps `precision` to the string length or to ≤6.
- **Impact / threat:** `strftime/3` documents accepting "any map … as long as they contain all of the relevant fields," and `microsecond` is typed `{non_neg_integer, non_neg_integer}` — so an out-of-range `precision` (e.g. `100`) is type-valid. An app that hand-builds a time-like map from untrusted `{value, precision}` and formats `%f` gets an uncaught `ArgumentError`. Bounded: a single controlled crash, no hang/memory blowup.
- **Evidence:** `verified` (re-executed this pass). `Calendar.strftime(%{microsecond: {5, 100}}, "%f")` → `RAISED ArgumentError … * 3rd argument: out of range`.
- **Suggested fix:** Clamp slice length, e.g. `binary_part(str, 0, min(byte_size(str), max(precision, 1)))`, or validate/clamp `precision` to `0..6` before slicing.

#### `CSCAN-calendar-strftime-calendar-dispatch-gadget` — Upgraded from hedged assumption in first pass
- **Finding ID:** `CSCAN-calendar-strftime-calendar-dispatch-gadget`
- **Severity:** Low
- **Confidence:** low
- **Location:** `calendar.ex:722` / `733` / `818` / `860` / `874` (`Date.day_of_week/day_of_year/quarter_of_year` calls), dispatching through `calendar/date.ex:915` (`calendar.day_of_week(year, month, day, starting_on)`).
- **Class:** Unsafe reflection / dynamic dispatch (bounded MFA gadget)
- **Issue:** `strftime/3` accepts "any map." Directives `%a/%A/%j/%q/%u` route through `Date.*`, which dispatch to `datetime.calendar.<fun>(...)` where `:calendar` is taken verbatim from the caller-supplied map. A caller that forwards a fully attacker-controlled map lets the attacker select which already-loaded module's callback is invoked (with attacker-controlled integer year/month/day args).
- **Impact / threat:** This is **not** a bytes→atom injection — the full call chain I examined (`calendar.ex` + `calendar/date.ex:914-916`) creates no atom from untrusted bytes; the module atom must already exist and be loaded. The realistic exposure is a bounded MFA gadget contingent on the caller passing the entire `datetime` map (including `:calendar`) from untrusted data — which the threat model classifies as code-supply, not data-injection. Hence low confidence / low severity; kept as a finding rather than a bare dismissal because I executed a proof that the dispatch fires.
- **Evidence:** `verified` that the dispatch fires (executed this pass): a module `EvilCal` with `day_of_week/4` placed in the map's `:calendar` field, then `Calendar.strftime(map, "%a", …)` → prints `EVIL EXECUTED` and returns `D1`. What remains *unverified* is any path that turns untrusted **bytes** into that module atom within Elixir's calendar stack — none exists in the two files read; producing the atom is the caller's responsibility.
- **Suggested fix:** None required at this layer if the documented contract ("caller supplies a trusted `:calendar` module") holds; optionally document that `strftime/3` must not be handed a map whose `:calendar` derives from untrusted input.

### Ruled out
- **`put_time_zone_database/1` (`calendar.ex:452`) — global application-env write.** Documented setter guarded by `is_atom/1` (`calendar.ex:452`); the only input is the caller-supplied atom, stored via `Application.put_env/3` (`calendar.ex:453`). No bytes→atom conversion in this file. Intended behavior. Basis: full read of the function body.
- **User formatter functions (`apply_format/3`, `calendar.ex:1031-1041`).** `am_pm_names`/`month_names`/etc. are caller-supplied functions; invoking a supplied function is code-supply, not data injection, and `apply_format/3` raises on non-arity-1/2 values (`calendar.ex:1039`). Basis: read of `options/1` (`calendar.ex:1043-1080`) and `apply_format/3`.
- **Infinite/exponential recursion in preferred formats.** `preferred_*_invoked` flags are set monotonically down each path (`calendar.ex:776`, `895`, `917`), capping depth at 3. Basis: `verified` — `preferred_datetime: "%c"` raises rather than recursing (re-executed this pass). The *multiplicative* blowup within that bounded depth is the amplification finding above.
- **Code execution / atom-from-bytes / file / command / deserialization / crypto sinks.** None present: no `Code.*`, `String.to_atom`, `Module.concat`, `File.*`, `System.*`, `:erlang.binary_to_term`, `Regex.compile`, or crypto calls. Basis: full read of the 1081-line file. (The calendar-dispatch case, previously folded into this dismissal as "an upstream concern outside this file's contract," is now split out as its own low-confidence finding above.)

## 3. Risks / caveats
- **Realism of the cubic case.** Full cubic amplification needs the attacker to control the format string *and* the three `preferred_*` option strings — documented user-facing config, but not the most common usage (format-string-only is ~170× linear). Medium reflects that the worst case is gated on options control while the linear case is broadly reachable; an app exposing preferred-format options to untrusted input should treat this as effectively High.
- **`%f` crash scope.** Needs a `microsecond` tuple whose `precision` exceeds its digit count. `Time`/`DateTime` constructors validate precision to `0..6`, so this bites only callers hand-building the map from untrusted data — hence Low.
- **Dispatch-gadget scope.** Requires the caller to forward a fully attacker-controlled map including `:calendar` set to an already-loaded module; no bytes→atom step exists in `calendar.ex` or `calendar/date.ex`. Kept at low confidence.

## Did I verify this?
- **Amplification finding:** verified by execution (re-run this pass): 400 B → 8 MB in ~919 ms; width cap and depth cap both fire as claimed.
- **`%f` crash:** verified by execution (re-run this pass): `microsecond: {5, 100}` raises `ArgumentError … 3rd argument: out of range`.
- **First-pass hedge "module-atom injection is an upstream concern outside this file's contract":** resolved. I read the neighbour `calendar/date.ex:914-916` and confirmed dispatch is `datetime.calendar.day_of_week(...)` on the caller-supplied module atom, and I *executed* a proof that a caller-map-controlled `:calendar` module's callback fires. Reading the neighbour told me what the dispatch does; the executed proof told me it is reachable. Neither reveals any bytes→atom step, so the residual classification (code-supply, out of scope) is stated as a low-confidence finding rather than a bare assumption.
- **First-pass hedge "I did not audit the downstream modules …":** narrowed. The downstream calls that matter for the reported sinks were checked — `Date.*` dispatch read directly (`calendar/date.ex`); `binary_part`/`String.pad_leading` behavior for `%f` confirmed by execution, not assumption. I did not exhaustively audit `DateTime.shift_zone!`, `NaiveDateTime.diff`, or `Integer.to_string`; no reported finding depends on their internals, and none introduces an untrusted-bytes→atom/exec sink reachable from `calendar.ex` that I could observe.

### Dropped in re-review
- None. All prior findings retained; the prior "behaviour/module dispatch" ruled-out entry was **split** — the formatter-function and setter portions remain ruled out (verified in-file), while the caller-map calendar-dispatch portion was **upgraded** to the low-confidence finding `CSCAN-calendar-strftime-calendar-dispatch-gadget` after executing a proof that the dispatch fires.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Code execution / atom creation / file / command / deserialization / crypto sinks.** None present — the file contains no `Code.*`, `String.to_atom`, `Module.concat`, `File.*`, `System.*`, `:erlang.binary_to_term`, `Regex.compile`, or crypto calls. Basis: full read of the 1081-line file.
