# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar.ex
<!-- sha256: 1d09a24467a01bd10b0f545ade312c69afc7dfee85af259e0dd221785401701f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:40:22Z_

I've thoroughly exercised the public surface (`strftime/3` across every format directive, padding/width paths, preferred-format recursion, `truncate/2`, the time-zone-database accessors, and the option validator) against the built compiler. Everything documented behaves correctly except one gap in the option validator.

## Summary

The `strftime/3` machinery is correct across all documented format directives and edge cases; the only defect found is that the option validator's whitelist accidentally includes three internal state flags, so it fails to reject them as unknown options.

## Bugfixes

#### `CSCAN-calendar-internal-strftime-flag-leak`
- **Finding ID** — `CSCAN-calendar-internal-strftime-flag-leak`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/elixir/lib/calendar.ex:1043-1080` (`options/1`), reached from `strftime/3:641`
- **Issue** — `options/1` validates user options by checking `Map.has_key?(acc, key)` against the *default options map*. That map also carries three internal state keys — `:preferred_datetime_invoked`, `:preferred_date_invoked`, `:preferred_time_invoked` (lines 1068-1070) — so passing any of them as a user option passes the `has_key?` check and overwrites the internal recursion-guard state instead of being rejected as an unknown option.
- **Class** — Contract violation (incomplete input validation; the documented "unknown option" rejection has a hole).
- **Impact** — The API promises `ArgumentError "unknown option ..."` for anything it does not recognize (line 1077). For these three keys that promise is silently broken: `Calendar.strftime(dt, "%c", preferred_datetime_invoked: true)` is accepted and then raises the misleading `"tried to format preferred_datetime within another preferred_datetime format"` at top level, even though no nesting occurred. A caller programmatically assembling the options keyword list (e.g. merging in a map) gets a confusing internal error or altered formatting behavior instead of the clear validation error the contract guarantees. Low severity: none of these are documented options, so the trigger is an out-of-spec key, but the defect is in the validator itself, which is reachable purely through the public `strftime/3` options argument.
- **Evidence** — `verified`. Ran:
  ```
  Calendar.strftime(~U[2019-08-26 13:52:06Z], "%c", preferred_datetime_invoked: true)
  #=> ** (ArgumentError) tried to format preferred_datetime within another preferred_datetime format
  Calendar.strftime(~U[2019-08-26 13:52:06Z], "%c", bogus: true)
  #=> ** (ArgumentError) unknown option :bogus given to Calendar.strftime/3
  ```
  The internal flag is silently accepted (no "unknown option" error) and corrupts guard state; a genuinely unknown key is correctly rejected.
- **Suggested fix** — Validate against an explicit set of the eight public option keys rather than the whole default map, e.g. keep a `@public_options` list (`[:preferred_date, :preferred_time, :preferred_datetime, :am_pm_names, :month_names, :day_of_week_names, :abbreviated_month_names, :abbreviated_day_of_week_names]`) and test `key in @public_options` in the reducer, or store the three `*_invoked` flags outside the option map entirely.

## Risks / caveats

- I verified the documented `%`-directive behaviors (`a A b B c d f H I j m M p P q s S u x X y Y z Z %`), padding/width handling (`pad_leading` uses `String.length`, `pad_leading_ascii` uses `byte_size`, applied only to ASCII-producing directives), negative-year sign placement, `%I`/`%p`/`%P` across all hour boundaries, `%f` precision truncation, `%z` offset arithmetic for positive/negative/fractional-hour offsets, `%s` epoch for both the DateTime and naive clauses, the preferred-format recursion guards (bounded to depth 3, no infinite loop), the `width > 1024` limit, invalid/trailing-`%` handling, and `truncate/2` — all match their docs/doctests. No further bug surfaced there.
- `strftime/3` on a map missing a field required by a directive (e.g. `%Y` on a `Time`) raises `KeyError`; this is the documented contract ("an error will be raised"), not a bug.
- `%s` on a non-UTC `DateTime` under the default `UTCOnlyTimeZoneDatabase` raises via `DateTime.shift_zone!/2`; that behavior originates in the `DateTime`/time-zone-database components, not this file.

### Ruled out
- **`%z`/`%y`/`%Y` arithmetic off-by-one** — checked +05:30, −05:30, +14:00, −03:30, −00:30, year −1/0, midnight/noon; all outputs correct. Basis: `verified`.
- **`pad_leading_ascii` mis-padding multibyte output** — every directive routed through `pad_leading_ascii` (`% d H I j m M q S u y Y z`) emits only ASCII (`Integer.to_string`, literal `%`, digit offsets); name/abbr/zone directives correctly use grapheme-aware `pad_leading`. Basis: `verified` (directive-by-directive audit of the format table).
- **Preferred-format infinite recursion** — the three `*_invoked` guards make each preferred directive invocable at most once along any path; cross-nesting (`%c`→`%x`→`%X`) terminates at the guard. Basis: `verified`.
- **`%f` truncation losing digits** — pad-to-6-then-take-`max(precision,1)` reproduces `.0`, `.048`, `.048531`; correct because `value` is total microseconds. Basis: `verified`.
- **Passing an internal `*_invoked` flag being "just spec misuse"** — considered folding the reported finding into this dismissal; rejected because the defect is in the validator's own whitelist (a documented safety feature), reachable through the public options argument, so it is reported rather than ruled out. Basis: `verified`.

