# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/time_zone_database.ex
<!-- sha256: eef5f454bc8ed5938bebfd12188fde06fe50e82eafc44fdaded28b09fea211c9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:46:56Z_

The behavior conforms exactly to the documented callback contracts. Both callbacks are total (catch-all second clauses), return shapes match the `@type time_zone_period` and `@callback` specs, and there are no doctests to fail.

# Summary
The file is a small behaviour definition plus a trivial UTC-only implementation; both callbacks are total and their return values conform to the documented `@callback` specs, so no genuine, reachable bug was found.

# Bugfixes

No findings

### Ruled out

- **Non-total function / missing clause** — Both `time_zone_period_from_utc_iso_days/2` and `time_zone_periods_from_wall_datetime/2` have a catch-all `(_, _)` second clause after the `"Etc/UTC"` clause, so every argument pair matches. Verified by calling with a non-UTC zone (`"Europe/London"`) and observing `{:error, :utc_only_time_zone_database}` rather than a `FunctionClauseError`. Basis: `verified`.
- **Contract / shape mismatch on returned map** — The `@type time_zone_period` requires keys `utc_offset`, `std_offset`, `zone_abbr` (plus optional any). The success clauses return exactly `%{std_offset: 0, utc_offset: 0, zone_abbr: "UTC"}`, matching the type; the error clauses return `{:error, :utc_only_time_zone_database}`, one of the two documented error atoms. Verified by inspecting the returned tuples/maps for both UTC and non-UTC inputs. Basis: `verified`.
- **`@impl` placement across multi-clause functions** — `@impl true` precedes the first clause of each callback implementation, which correctly annotates all clauses of that function head; no `@impl`-on-non-callback or missing-`@impl` warning. Confirmed by compiling the module (loaded and invoked without warnings about behaviour implementation). Basis: `verified`.
- **Ambiguous/gap paths** — The `UTCOnlyTimeZoneDatabase` never returns `:ambiguous` or `:gap` because UTC has no DST; returning plain `{:ok, period}` for all wall datetimes in `Etc/UTC` is correct and matches the behaviour's allowance of `{:ok, time_zone_period}` for a single possible period. Basis: `asserted` (semantics of a fixed-offset zone: no transitions can produce ambiguity or gaps) plus `verified` that the `Etc/UTC` clause returns `{:ok, ...}`.

# Risks / caveats
The file contains no parsing, arithmetic, Unicode handling, or process/state logic — the surface where the reviewed bug classes typically live is absent — so the review reduced to checking callback totality and return-shape conformance, both of which hold.

