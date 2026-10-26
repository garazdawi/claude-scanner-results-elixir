# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/iso.ex
<!-- sha256: 212f14202f1fa45ecb3e1c3b1ae7b97e6678882179e7bc86142462eb22f3f17f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:46:53Z_

I've done a thorough review: read the entire file, ran all 64 doctests (all pass), and exercised the tricky paths (offset day-rollover, negative years, fractional-second durations, leap-year validation, weekday offsets) against the built compiler. Here is my report.

## Summary
After enumerating candidate defects across the parsers, formatters, arithmetic helpers, and shift functions and verifying the suspicious paths against the built Elixir (`1.21.0-dev`), I found no genuine, reachable bug in this file's documented public API.

## Bugfixes

No findings

### Ruled out

- **`parse_utc_datetime` day-rollover drops microseconds / uses stale date** (lines 671–684): the `case` rebinds `year, month, day, hour, minute, second`, and microseconds are unaffected because offsets are whole seconds. Basis: `verified` — `parse_utc_datetime("2015-01-23 00:50:07.123456+02:30")` → `{:ok, {2015, 1, 22, 22, 20, 7, {123456, 6}}, 9000}` and `"...23:50:07.999999-02:30"` → `{:ok, {2015, 1, 24, 2, 20, 7, {999999, 6}}, -9000}`, both dates/times/microseconds correct.
- **Fractional-second duration sign loss** (lines 734–750): `Integer.parse("-0.5S")` yields `0`, losing the sign, but the explicit `case string do "-" <> _ -> -ms` branch recovers it. Basis: `verified` — `parse_duration("PT-0.5S")` → `{:ok, [second: 0, microsecond: {-500000, 1}]}`; `"PT-1.5S"` → `{second: -1, microsecond: {-500000, 1}}`.
- **`day_of_week_offset` table off-by-one across `starting_on` values** (lines 1112–1119): all eight offsets are internally consistent. Basis: `verified` — computed each `starting_on` against the Monday date 2016‑10‑31 and every result matches (`:monday`→1, `:sunday`→2, `:saturday`→3, `:tuesday`→7, `:wednesday`→6, `:thursday`→5, `:friday`→4), consistent with the passing doctests.
- **`shift_months` negative-year month wraparound** (lines 1911–1926): `rem` can go negative, but the `new_month < 1 -> new_month + 12` guard plus `floor_div_positive_divisor` correct it. Basis: `verified` — `shift_date(-1, 12, 31, Duration.new!(day: 1))` → `{0, 1, 1}`.
- **`zero_pad` unhandled pad-width / negative-year formatting** (lines 1743–1758): the `case` covers pad widths 0–5, and `count - byte_size` never exceeds 5 for any documented call (year ≤ 4 pad, micro ≤ 6 pad with ≥1 digit). Basis: `verified` — full doctest run passes, incl. `date_to_string(-99,1,31)` → `"-0099-01-31"`.
- **`-00:00` / `+00:00` offset handling in `parse_offset`** (lines 2096–2107): `-00:00` is deliberately rejected (negative-zero offset) while `+00:00` and `Z` yield offset `0`; this is intentional, not a defect. Basis: `verified` — `"+00:00"` → `{:ok, …, 0}`, `"-00:00"` → `{:error, :invalid_format}`.
- **Fractional/trailing-garbage acceptance in duration & date/time parsers** (lines 736, 438–450): fixed-width date match (no trailing `rest`), the `"S"`-exact remainder check, and `find_date_unit`/`find_time_unit` returning `false` all reject malformed input. Basis: `verified` — `"P1.5D"`, `"PT1.S"`, `"PT"`, `"P5"` and 9/11-byte dates all return the appropriate `:error` tuples in spot checks.
- **Doc/code mismatch (any doctest)**: Basis: `verified` — `doctest Calendar.ISO` reports `64 passed`.

## Risks / caveats
- I scoped strictly to the documented public API; `@doc false` exported helpers (`date_to_iso_days`, `date_from_iso_days`, `add_time_unit_to_iso_days`, `from_unix`, `iso_days_to_unit`, etc.) were checked only insofar as documented functions reach them. A defect reachable *only* by calling those directly would be out of scope and is not covered here.
- Crashes on arguments that violate a function's declared type/`@spec` (e.g. `parse_time("x", :foo)`, `days_in_month(y, 13)`, out-of-range microseconds into `time_to_iodata`) were treated as contract misuse, not bugs, per the review scope.
- The very-large / extreme-value regime of the Neri–Schneider date arithmetic (`valid_date_to_iso_days` / `date_from_iso_days`) was validated only within the `-9999..9999` year range enforced by the public parsers, matching the documented `year` type.

