# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/date.ex
<!-- sha256: 53ffb5eb1e9fe21ea197ec60d0d9f061e5ef7486649f9ce4116123d07bfe3378 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/date.ex
<!-- sha256: 53ffb5eb1e9fe21ea197ec60d0d9f061e5ef7486649f9ce4116123d07bfe3378 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed 2026-08-28. The file under review is not physically present in the scanner tree (`.../src/` contains only the `.erl` front-end sources); the reviewable copy is the installed `/opt/elixir/lib/elixir/lib/calendar/date.ex` (Elixir 1.21.0-dev, Erlang/OTP 29). Every result below was produced by running that exact source through the sandbox toolchain, not asserted from an invariant._

## Summary

No genuine, caller-reachable defects found in `Date`. This module is almost entirely thin delegators to `calendar.*` callbacks; its own logic (range construction, `add/2`, the week-boundary destructuring, and the `Inspect` clause ordering) was driven through both the ISO and a real non-ISO calendar and matched in every case.

The single hedge the first-pass reviewer flagged — "did not exhaustively drive every doctest" — is discharged here by **my own execution**, not by the prior narrative: I ran the module doctests twice and reproduced the results below.

## Bugfixes

No findings.

## Risks / caveats

- **Scope boundary (not a hedge).** `leap_year?`, `days_in_month`, `day_of_year`, `year_of_era`, `day_of_era`, `quarter_of_year`, `months_in_year`, `day_of_week/2`, and `shift/2` forward directly to the calendar module (`Calendar.ISO` or a third-party calendar). A defect in those callbacks would surface in that file, not here. This is a genuine scope boundary — the delegating clauses in *this* file are trivially correct given the callback contract, which I confirmed by execution for the non-trivial consumers (`add/2`, week boundaries).

## Did I verify this

- **Full doctest suite (52 tests) — run by me.** `doctest Date` against the installed source: first run `50/52` passed, the 2 failures being solely `UndefinedFunctionError`/`module Calendar.Holocene is not available` for the `convert/2` and `convert!/2` examples (the Holocene calendar is test-support, not shipped). I then `Code.require_file`'d the real `test/elixir/calendar/holocene.exs` support module and re-ran: **52/52 passed.** This discharges the flagged hedge with executed evidence.
- **Non-ISO `beginning_of_week/2` / `end_of_week/2` — run by me.** Drove `Calendar.Holocene` (a real non-ISO calendar, ISO-offset by 10 000 years) at `12020-07-11` (≡ ISO `2020-07-11`) through both functions for `starting_on ∈ {:default, :sunday, :saturday}` and compared to the ISO struct at the same instant. All six pairs matched: bow → `07-06 / 07-05 / 07-11`; eow → `07-12 / 07-11 / 07-17`. Both triple-destructure clauses fired: the equal-bound short-circuit (`{d,d,_}` for bow `:saturday`; `{d,_,d}` for eow `:sunday`, each returning the input unchanged) and the `add/2`-shift clause (all other cases).
- **Non-ISO `add/2` — run by me.** `Date.add(%Date{calendar: Calendar.Holocene, year: 9990, month: 1, day: 1}, -2)` (≡ ISO `-0010-01-01`) → `-0011-12-30` rendered in `Calendar.Holocene`, identical shift to `Date.add(~D[-0010-01-01], -2)` → `~D[-0011-12-30]`, original calendar preserved. Exercises the non-ISO clause (lines 728–731) across a year boundary.
- **`Inspect` clause selection — run by me.** `inspect` of the Holocene value at year `12000` (outside `-9999..9999`) rendered `~D[12000-01-01 Calendar.Holocene]` (clause 1, sigil form); `inspect` of a `Calendar.ISO` struct at year `10000` (outside range) rendered `Date.new!(10000, 1, 1)` (clause 2). Confirms clause 1's guard catches every non-ISO struct.

### Ruled out

- **`Inspect` third clause (line 1187) is unreachable dead code.** Clause 1's guard `calendar != Calendar.ISO or year in -9999..9999` is true for *every* non-`Calendar.ISO` struct (first disjunct), so clause 1 matches all non-ISO input; clause 2 matches all remaining `Calendar.ISO` structs. The non-ISO `Date.new!(.., calendar)` fallback at 1187 is therefore never selected. Not caller-facing: every input still renders correctly (non-ISO via clause-1 sigil form, ISO-out-of-range via clause-2 `Date.new!` form). Basis: `verified` (guard tautology + observed renders — I ran both the year-12000 Holocene case → sigil form and the year-10000 ISO case → `Date.new!` form). Confidence: **High**. Axes examined: calendar ≠ ISO with year outside -9999..9999; ISO with year outside -9999..9999 | Not examined: none.
- **`range/2` negative-range path (lines 112–121)** emits `IO.warn` and infers step `-1` rather than raising. Documented, intentional deprecation nudge toward `range/3` (doc text lines 82–84; matching warning string at 116–118). Basis: `verified` (doc contract + doctest suite exercises `range/2` and `range/3` and passed in my run). Confidence: **High**. Axes examined: increasing range, doc contract, doctest execution | Not examined: none material.
- **`add/2` doc note "always considers a day … according to `Calendar.ISO`"** — the non-ISO clause (728–731) round-trips through `to_iso_days`/`from_iso_days`, so the shift is ISO-Gregorian and the result is returned in the original calendar, as documented. Basis: `verified` (executed non-ISO clause across a year boundary; result matched ISO and preserved `Calendar.Holocene`). Confidence: **High**. Axes examined: non-ISO calendar, negative offset, year-boundary crossing | Not examined: none material.
- **`beginning_of_week/2` / `end_of_week/2` non-ISO clauses** correctly destructure the `{day_of_week, first, last}` triple from `calendar.day_of_week/4` and shift via `add/2`; the equal-bound clauses (`{d,d,_}` / `{d,_,d}`) short-circuit the no-op case. Basis: `verified` (executed — both the short-circuit and the `add/2`-shift clause fired across all three `starting_on` values and agreed with ISO in all six pairs). Confidence: **High**. Axes examined: `starting_on ∈ {default, sunday, saturday}`; equal-bound short-circuit clause; `add/2`-shift clause | Not examined: none.

### Dropped in re-review

No prior findings dropped (the first pass reported none). No ruled-out entry dropped or downgraded. The four ruled-out entries were re-verified against my own execution rather than the prior report's narrative; each retains `verified` basis with `Not examined: none`/`none material`, every listed axis actually run this pass (doctest suite, non-ISO week boundaries, non-ISO `add`, both `Inspect` selection axes). No hedged assumption remained that required upgrading to a low-confidence finding — the only flagged hedge ("did not exhaustively drive every doctest") was discharged by re-execution (52/52).
