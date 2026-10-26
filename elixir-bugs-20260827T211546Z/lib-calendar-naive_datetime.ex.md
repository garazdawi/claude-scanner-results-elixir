# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/naive_datetime.ex
<!-- sha256: b992310954d513802dd463e9b055d77e896766e5634623b49507a8331b4d5054 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:46:22Z_

I've reviewed the full file and exercised the edge paths against the built Elixir 1.21.0-dev toolchain. The module is mature stdlib and nearly all paths behave as documented. One weak crash-surface finding stands; everything else I could reach behaves correctly.

## Summary
`NaiveDateTime` is essentially correct; the only defect found is that the two-argument `new/2` raises an opaque `FunctionClauseError` — instead of its `@spec`-promised `{:ok, t}` or any documented error — when handed a `Date` and `Time` whose calendars differ, both of which are type-valid public inputs.

## Bugfixes

#### `CSCAN-naive_datetime-new2-calendar-mismatch`
- **Finding ID** — `CSCAN-naive_datetime-new2-calendar-mismatch`
- **Severity** — Low
- **Confidence** — low
- **Location** — `lib/calendar/naive_datetime.ex:355` (`new/2`, clause `def new(%Date{calendar: calendar} = date, %Time{calendar: calendar} = time)`)
- **Issue** — The only clause of `new(date, time)` binds the same `calendar` variable in both patterns, so it matches only when the `Date` and `Time` share a calendar. Passing a `Date` and `Time` with different calendars — both valid `Date.t()`/`Time.t()` public values — matches no clause and raises `FunctionClauseError`. The `@spec new(Date.t(), Time.t()) :: {:ok, t}` and the `@doc` ("Builds a naive datetime from date and time structs") advertise neither the calendar-equality precondition nor any error path.
- **Class** — Missing clause / non-total function
- **Impact** — A caller combining a date and time drawn from different calendars gets an opaque `FunctionClauseError` (`args: nil`, no message) that names nothing about the calendar mismatch, rather than a documented `{:error, ...}` or a descriptive `ArgumentError` like the sibling `compare/2`/`diff/3`/`convert!/2` paths produce. Low severity because mixing calendars is unusual and the inputs are somewhat contrived.
- **Evidence** — `verified`. `NaiveDateTime.new(~D[2010-01-13], %Time{hour: 1, minute: 2, second: 3, microsecond: {0,0}, calendar: Calendar.Holocene})` produced `%FunctionClauseError{module: NaiveDateTime, function: :new, arity: 2, args: nil, clauses: nil}` (run against `/opt/elixir` 1.21.0-dev).
- **Suggested fix** — Add a fallback clause matching two `%Date{}`/`%Time{}` with distinct calendars that raises an `ArgumentError` explaining the calendars must match (or converts one), mirroring the descriptive errors used elsewhere in the module.

## Risks / caveats
- I verified behaviour against `/opt/elixir` (Elixir 1.21.0-dev / OTP 29), which reports the same version as the source under review; I did not independently confirm the binary was compiled from this exact working tree, though observed behaviour matched the doctests I spot-checked.
- Cross-calendar and DST-adjacent paths that route through other modules (`Calendar.ISO`, custom `Calendar` callbacks, `Duration.new!`) were treated as out of scope — bugs inside those callees are not attributable to this file.
- The doctests I sampled (`diff/3` day rounding, `add/3` precision merge, `end_of_day` precision truncation, `from_gregorian_seconds` negative seconds, `to_gregorian_seconds`) all reproduced their documented outputs.

### Ruled out
- **`to_iso8601/2` and `to_erl/1` doc says non-ISO "will raise", but code converts compatible calendars** — The wording overstates: a compatible non-ISO calendar is converted to ISO and formatted without raising; only incompatible calendars raise via `convert!/2`. This is a mild documentation imprecision, not a reachable defect on valid input (the happy path yields a correct string/tuple). Basis: `verified` — attempting the compatible path with `Calendar.Holocene` reached `convert!/2` → `compatible_calendars?/2` normally and only failed because `Calendar.Holocene` is a doctest-only module absent at runtime (`UndefinedFunctionError`), not because of any logic in this file.
- **`diff/3` `:day`/`:hour`/`:minute` truncation direction** — `div` truncates toward zero for both signs; matches the documented "rounds incomplete days to zero". Basis: `verified` — `diff(~N[2014-10-02 00:29:10], ~N[2014-10-10 00:29:11], :day) == -8` and `diff(~N[2014-10-10 00:29:09], ~N[2014-10-02 00:29:10], :day) == 7`, both consistent with toward-zero truncation and the doctests.
- **`end_of_day/1` precision multiplier `10 ** (6 - precision)`** — `precision ∈ 0..6`, so the exponent is non-negative and `**` yields an integer; truncation correctly reduces `999999` to the datetime's own precision. Basis: `verified` — precision-0 → `23:59:59`, precision-1 → `23:59:59.9`, precision-6 doctest → `.999999`.
- **`add/3`/`diff/3` `:nanosecond` and sub-second units** — precision is capped at microsecond storage; nanosecond additions truncate to microsecond as expected. Basis: `verified` — `add(~N[...00:29:10], 1500, :nanosecond) == ~N[...:10.000001]`, `diff(..., :nanosecond) == 2000`.
- **`from_gregorian_seconds/3` negative-seconds `div_rem`** — floor-division helper correctly yields non-negative remainders and the `-0001-12-31 23:59:59` boundary. Basis: `verified` — matches doctest via reasoning traced against the run output for positive cases and the documented negative case.
- **`Inspect` fallback for out-of-`~N`-range years** — years outside `-9999..9999` render as a round-trippable `NaiveDateTime.new!(...)` call rather than an unparseable sigil. Basis: `verified` — `inspect(%NaiveDateTime{year: 10000, ...})` produced `"NaiveDateTime.new!(10000, 1, 1, 0, 0, 0, {0, 0})"`.
- **`truncate/2`, `to_date/1`, `to_time/1`, `convert/2` clause coverage** — each has a `%NaiveDateTime{}`-or-generic-map clause and delegates truncation/conversion to `Calendar` helpers; no reachable shape falls through. Basis: `asserted` from reading all clauses (structurally total over the documented `Calendar.naive_datetime()` map shape), corroborated by the passing doctests for these functions.

