# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/datetime.ex
<!-- sha256: 9de34ed2985f7b3684550ea61a75462698325f620ec1f7b7c32e43195a94fe0f -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/datetime.ex
<!-- sha256: 9de34ed2985f7b3684550ea61a75462698325f620ec1f7b7c32e43195a94fe0f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T00:00:00Z (hedge-resolution pass over the 2026-08-27 first pass)_

Both first-pass findings survive and were re-reproduced by execution on the sandbox Erlang/OTP 29 + Elixir 1.21.0-dev toolchain. The ruled-out entries now carry `Axes examined | Not examined` fields; three of the four were narrowed to `Not examined: none` by pushing the untried axes through the real implementation. The first-pass "out of scope" scoping caveat is restated below as a plain scope boundary (no behavioural claim rides on it).

## Summary
`DateTime`'s `from_map/1` helper silently drops the `:calendar` field, so `convert/2` and `truncate/2` corrupt non-ISO datetime maps to `Calendar.ISO`, and `from_unix!/3` raises an opaque `CaseClauseError` (instead of a clear `ArgumentError`) when given an ISO-incompatible calendar.

## Bugfixes

#### `CSCAN-datetime-from-map-drops-calendar`
- **Finding ID** — `CSCAN-datetime-from-map-drops-calendar`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `lib/calendar/datetime.ex:2059-2073` (`from_map/1`), reached via `convert/2` clause at `:1960-1962` and `truncate/2` clause at `:1925-1927`.
- **Issue** — `from_map/1` reconstructs a `%DateTime{}` copying every field *except* `:calendar` (confirmed by re-reading lines 2059-2073: the struct literal lists `year` through `std_offset`, no `calendar` key), so the result takes the struct default `Calendar.ISO`. Both the generic-map clause of `convert/2` (`convert(%{calendar: calendar} = datetime, calendar)`, line 1960) and the generic-map clause of `truncate/2` (line 1925) route through it. A `Calendar.datetime()` map in a non-ISO calendar therefore has its calendar silently reset to `Calendar.ISO` while its calendar-specific `year/month/day` numbers are preserved verbatim — a semantically wrong datetime.
- **Class** — State / accumulator threading (a field dropped across a struct rebuild).
- **Impact** — A caller holding datetimes as plain maps in a third-party/custom calendar (the exact reason `convert/2` and `truncate/2` are typed to accept `Calendar.datetime()` rather than `t()`) gets back a `DateTime` mislabeled as ISO. `convert(map, its_own_calendar)` — an identity conversion — returns the wrong calendar; every downstream calendar-aware operation then misinterprets the date fields. Silent corruption, no error raised.
- **Evidence** — `verified` (re-run in this pass):
  ```
  $ elixir t.exs   # m = %{calendar: FakeCal, year: 2000, month: 2, day: 29, ...}
  convert calendar: Calendar.ISO      # expected FakeCal
  truncate calendar: Calendar.ISO     # expected FakeCal
  ```
- **Suggested fix** — Add `calendar: datetime_map.calendar` to the struct built in `from_map/1`.

#### `CSCAN-datetime-from-unix-bang-caseclause`
- **Finding ID** — `CSCAN-datetime-from-unix-bang-caseclause`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/calendar/datetime.ex:477-485` (`from_unix!/3`).
- **Issue** — `from_unix/3` calls `convert(iso_datetime, calendar)` (line 443), which can return `{:error, :incompatible_calendars}`. `from_unix!/3`'s `case` (lines 478-484, re-read this pass) matches only `{:ok, datetime}` and `{:error, :invalid_unix_time}`, so any other error tuple falls through with no clause and raises `CaseClauseError`.
- **Class** — Error-handling asymmetry / missing clause.
- **Impact** — A caller passing a valid but ISO-incompatible `Calendar.calendar()` (different day-rollover moment) to `from_unix!/3` gets a bare `(CaseClauseError) no case clause matching: {:error, :incompatible_calendars}` naming nothing actionable, instead of the clear `ArgumentError` the sibling error path produces. The non-bang `from_unix/3` correctly returns the error tuple, so the failure is limited to the bang variant.
- **Evidence** — `verified` (re-run in this pass): with `IncompatCal.day_rollover_relative_to_midnight_utc/0 = {12, 24}`:
  ```
  compatible? false
  from_unix/3:  {:error, :incompatible_calendars}
  from_unix!/3 raised: CaseClauseError: no case clause matching: {:error, :incompatible_calendars}
  ```
- **Suggested fix** — Add a catch-all `{:error, reason}` clause to the `case` that raises `ArgumentError` with the reason (mirroring the other bang builders in this module, e.g. `new!/4`).

## Risks / caveats
- Both findings are on interop paths reachable only with non-ISO / custom calendars; callers using only `Calendar.ISO` (the overwhelming default) never hit them, which is why they've gone unnoticed. They are nonetheless within the documented `Calendar.datetime()` / `Calendar.calendar()` contracts of public functions.
- Scope boundary (not an unverified assumption): I reviewed the full 2160-line file against the Phase-1 classes, including this file's *own* logic for `to_iso8601` (the `Z`-vs-offset guards at `:1124-1178`, `shift_by_offset/2` at `:1199-1207`), `diff` (`:1603-1613`), and the clause routing of `convert`/`truncate`. The actual date/offset **arithmetic** those functions call — `Calendar.ISO.*` and `System.convert_time_unit/3` — lives in other files and is out of this file's review scope; no behavioural claim in this report depends on it. The Unicode/UAX/UTS clause checklist in the standing context governs the tokenizer/unicode sources, not this calendar file, and does not apply here.

### Ruled out

- **`from_iso8601` `-00:00` returning `:invalid_format`** (`:1265-1266`) — the rejection is produced inside `Calendar.ISO.parse_utc_datetime`, not this file; rejecting negative-zero offset is the documented ISO 8601 behaviour (`-00:00` is not a legal offset there, unlike RFC 3339). Basis: `verified`.
  - Axes examined: offset sign (`+00:00` → `{:ok, …, 0}`; `-00:00` → `{:error, :invalid_format}`; `Z` → `{:ok, …, 0}`; `-00:30` → `{:ok, …, -1800}`, executed this pass) | Not examined: none — the full "which offset spellings are rejected" axis was driven through the real parser and only the negative-zero spelling is rejected.

- **`to_iso8601/3` with `offset: 0` on a non-UTC zone emitting `+00:00` rather than `Z`** — intended: the `Z` clause (`:1158-1178`) is guarded on `time_zone: "Etc/UTC"`, and only that exact zone yields `Z`. Basis: `verified`.
  - Axes examined: time_zone value crossed with `offset: 0` — `America/Manaus` (offset 0) → `+00:00`, `Etc/GMT` (a non-`Etc/UTC` zone with total offset 0) → `+00:00`, `Etc/UTC` → `Z` (executed this pass) | Not examined: none — the `Z`-vs-`+00:00` decision is fully determined by the `time_zone == "Etc/UTC"` guard, exercised on both sides of the guard.

- **`diff/3` `:day/:hour/:minute` truncation direction** (`:1603-1613`) — `div/2` truncates toward zero, which the `@doc` explicitly promises ("Fractional results are not supported and are truncated"). Basis: `verified`.
  - Axes examined: sign of the fractional result — `+1.5 day` → `1`, `-1.5 day` → `-1` (executed this pass, confirming truncate-toward-zero rather than floor in both directions) | Not examined: none.

- **`shift_zone/3` same-zone shortcut** (`:741-743`) returning the datetime unchanged — correct; when the target `time_zone` already equals the datetime's, no offset/wall-time recomputation is defined to occur. Basis: `asserted`.
  - Axes examined: matching `time_zone` (the head `shift_zone(%{time_zone: tz} = dt, tz, _)` returns `{:ok, dt}` verbatim) | Not examined: stored-offset-consistency axis — whether a same-zone datetime carrying a stale `utc_offset`/`std_offset` should be re-derived. I could not drive this axis: `shift_zone` has no reference implementation that recomputes on a same-zone input to disagree with, so there is no differential input to run. Entry stays `asserted`.

## Did I verify this
- **`from_map` drops `:calendar`** — Yes. Read `from_map/1` (`:2059-2073`) and both call sites (`:1925-1927`, `:1960-1962`); re-executed the reproduction — `convert` and `truncate` both return `Calendar.ISO` for a `FakeCal` map.
- **`from_unix!` `CaseClauseError`** — Yes. Read the `case` (`:477-485`) and the `from_unix/3` `convert` call (`:443`); re-executed with an incompatible calendar — `from_unix/3` returns `{:error, :incompatible_calendars}`, `from_unix!/3` raises `CaseClauseError`.
- **Ruled-out 1/2/3** — Yes, by execution (see axes above); all narrowed to `Not examined: none`.
- **Ruled-out 4 (`shift_zone` shortcut)** — Partially: verified the code path by reading `:741-743`; the stored-offset-consistency axis could not be driven (no differential reference), so it remains `asserted`, not `verified`.

### Dropped in re-review
- None. Both first-pass findings are preserved and re-confirmed; no ruled-out entry was upgraded to a finding, and no finding was dropped.

### Upgraded from hedged assumption in first pass
- None. The first pass's only hedge was the "out of scope" scoping sentence in Risks/caveats. Re-reading the file showed this file's own logic for the delegating functions is limited to guards/routing/offset-shifting (all reviewed, no defect), and the underlying arithmetic genuinely resides in separate files (`Calendar.ISO`, `System`). This is a true scope boundary, not an unverified behavioural assumption, so no low-confidence finding is warranted; it is restated plainly rather than upgraded.
