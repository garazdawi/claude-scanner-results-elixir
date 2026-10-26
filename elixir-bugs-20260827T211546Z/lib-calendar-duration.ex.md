# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/duration.ex
<!-- sha256: d7b3ed9db21577f200a4d8bbe62e635aa1186c3194624b694304f248badf98c0 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:43:29Z_

## Summary
After exhaustively exercising every documented public entry point (`new!/1`, `add/2`, `subtract/2`, `multiply/2`, `negate/1`, `from_iso8601/1`, `from_iso8601!/1`, `to_string/2`, `to_iso8601/1`) — including sign boundaries, microsecond carry/overflow, ordering, precision‑0, and all validation/error paths — I found no genuine reachable bug in this file.

## Bugfixes
No findings

### Ruled out

- **`second_component/3` sign handling for negative fractional seconds** — The `div`/`rem`/`abs` + explicit `sign` construction was the most suspect spot (truncation toward zero, "-0.x" cases, mixed-sign second/microsecond). Tested `{1,-1200000,4}→"-0.2000s"`, `{-1,200000,6}→"-0.800000s"`, `{2,-1500000,4}→"0.5000s"`, `{0,-999999,6}→"-0.999999s"`, `{-1,-500000,6}→"-1.500000s"`. All correct; sign never doubled or dropped. Basis: `verified` (ran each case, output matches expected magnitude+sign).
- **`to_string/2` component ordering (double-reversal of accumulator + `Enum.reduce/2`)** — The acc is built smallest-unit-first and `reduce/2` re-reverses it; suspected an ordering inversion. Verified full struct renders `"1a 2mo 3wk 4d 5h 6min 7s"` and negatives `"-1a -7.000005s"` in correct magnitude order with correct separator placement. Basis: `verified`.
- **Microsecond overflow not carried into seconds** — `add/2` can produce `microsecond: {1_200_000, 6}`; suspected malformed output. `second_component/3` recomputes `total_ms` and correctly carries: `add({600000,6},{600000,6})` → `"PT1.200000S"`. Basis: `verified`.
- **`validate_unit!/1` clause ordering (microsecond tuple vs generic integer vs unknown unit)** — Checked the five-clause ladder can't let an invalid pair through or wrongly reject a valid one. `foo:1`, `microsecond: {1.0,3}`, `day: 1.5` each raise the correct documented `ArgumentError`; valid pairs pass. Basis: `verified`.
- **`from_iso8601/1` and `from_iso8601!/1` contract shapes** — `from_iso8601("bad")` returns `{:error, :invalid_duration}`; the bang version raises `ArgumentError` with the documented message. No `CaseClauseError` on the `{:ok,_} | {:error,_}` match for valid `Calendar.ISO.parse_duration/1` outputs. Basis: `verified`.
- **All 24 module doctests** — Ran the module through `ExUnit` `doctest Duration`; 24 passed, 0 failed, so no code/doc contract mismatch exists in any documented example. Basis: `verified` (`Result: 24 passed`).

## Risks / caveats
- `to_string/2`/`to_iso8601/1` with `microsecond: {n, 0}` where `n ≠ 0` silently drops the fractional value (e.g. `{-500000, 0}` renders `"0s"`). This is consistent with "precision 0 = no fractional digits displayed" and is not reachable from typical calendar-derived durations, so I did not treat it as a defect — but it is a value-lossy edge if a caller hand-builds such a struct.
- `from_iso8601/1`'s totality depends on `Calendar.ISO.parse_duration/1` (a different file) only ever returning `{:ok,_}` or `{:error, atom}`; a shape change there would surface as a `CaseClauseError` here. That contract lives outside the reviewed file and is out of scope.

