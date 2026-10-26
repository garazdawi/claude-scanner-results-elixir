# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/iso.ex
<!-- sha256: 212f14202f1fa45ecb3e1c3b1ae7b97e6678882179e7bc86142462eb22f3f17f -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/iso.ex
<!-- sha256: 212f14202f1fa45ecb3e1c3b1ae7b97e6678882179e7bc86142462eb22f3f17f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

# Security Review — `lib/elixir/lib/calendar/iso.ex`

## Summary
`Calendar.ISO` is a pure date/time calendar and ISO 8601 parser/formatter with no code-exec, filesystem, network, atom-creation, or dynamic-dispatch sinks. The only meaningful issue remains that the duration parser does not bound numeric-component length, so a large untrusted string raises an uncaught `SystemLimitError` instead of the documented error tuple. In this re-review I executed the previously-hedged cross-boundary flow (untrusted string → `Duration.from_iso8601` → `Date.shift` → in-file bignum date arithmetic) and confirmed the arithmetic itself is fast and non-crashing, so the "bignum blow-up" concern stays ruled out — now on executed evidence, not on an "outside this file" assumption.

## Security

#### `CSCAN-iso-parse-duration-unbounded-integer`
- **Finding ID:** `CSCAN-iso-parse-duration-unbounded-integer`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/elixir/lib/calendar/iso.ex:722` (`parse_duration_date/4`) and `:733` (`parse_duration_time/4`); entry point `parse_duration/1` at `:698`.
- **Class:** Resource consumption (unbounded parse → uncaught exception / documented-contract bypass)
- **Issue:** `parse_duration/1` feeds each numeric component straight into `Integer.parse/1` (`iso.ex:722`, `:733`) with no limit on the number of digits. For a component of roughly 1.3 million digits or more, the underlying `:erlang.binary_to_integer/2` raises `SystemLimitError`. That exception is raised *inside* the `with … do … else …` expression, and an `else` block only rescues pattern-match failures of the `<-` clauses — it does **not** catch a raised exception. The error therefore propagates out of `parse_duration/1` (and out of `Duration.from_iso8601/1`), even though both are specced to return `{:ok, …} | {:error, atom}`.
- **Impact / threat:** A caller that parses an ISO 8601 duration from untrusted data — e.g. `Duration.from_iso8601(params["duration"])` in a request handler, an iCalendar/feed parser, or any API accepting durations — and relies on the documented `{:error, _}` result to reject bad input will instead have an exception thrown past its error handling. A single ~1.3 MB string (within typical HTTP body limits) crashes the parsing process. Trust boundary: untrusted string → documented public parser. Contained by process supervision; no memory corruption or escalation, hence Low.
- **Evidence:** `verified` (re-confirmed this pass, OTP 29 / Elixir 1.21.0-dev):
  - `Calendar.ISO.parse_duration("P" <> String.duplicate("9", 1_300_000) <> "Y")` → `{:RAISED, SystemLimitError}` (input 1,300,002 bytes).
  - Same raise through the higher-level API: `Duration.from_iso8601("P" <> 1_300_000×"9" <> "D")` → `{:RAISED_PARSE, SystemLimitError}`.
  - Below the threshold the call returns normally: 100k/500k/1M/1.2M-digit components all parse (`{:ok, …}`), with cost concentrated in `Integer.parse` (3 ms / 29 ms / 83 ms / 131 ms total).
- **Suggested fix:** Bound the accepted digit run before conversion in `parse_duration_date/4` and `parse_duration_time/4`, returning `{:error, :invalid_date_component}` / `{:error, :invalid_time_component}`; or wrap the `Integer.parse/1` conversion so a `SystemLimitError` becomes the documented error tuple.

### Ruled out

- **Bignum blow-up in date arithmetic (`date_to_iso_days`, `shift_date`/`shift_days`/`shift_months`, `date_from_iso_days`, `valid_date?`)** — **hedge "outside this file's untrusted-input surface" removed; verified by execution.** In the first pass this was dismissed on the reasoned assumption that oversized integers only arise from caller-hand-built structs. That assumption was incomplete: `parse_duration/1` *does* emit unbounded integers (see finding above), and those integers can reach the in-file `shift_*` functions through the two-call flow `Duration.from_iso8601/1` → `Date.shift/2` → `Calendar.ISO.shift_date/4` (`iso.ex:1816`) → `shift_days/2` (`iso.ex:1902`) → `date_from_iso_days/1`. So oversized integers **can** reach this file's date arithmetic from untrusted input. I pushed that flow through and measured the consequence rather than assuming it:
  - `Duration.from_iso8601("P100000000000000000000000000D")` (27-digit day) → `Date.shift(~D[2000-01-01], d)` → `Date.new!(273790700698850763535816, 7, 29)` in 0 ms; the analogous 27-digit month value → `shift_months` → valid Date in 0 ms.
  - Scaling the day component to 100k / 500k / 1M / 1.2M digits: `from_iso8601` succeeds and the **`Date.shift` step itself runs in ≤1 ms** in every case (total time is spent in the parser, not the arithmetic). Bignum division in `date_from_iso_days` is effectively linear in digit count and never crashes or overflows.
  - Beyond ~1.3M digits, `from_iso8601` raises `SystemLimitError` *before* any Date is produced, so the shift path is never reached — it is capped by the same integer limit already captured in the finding.
  - Parse-boundary bound re-confirmed by reading `iso.ex:227-241`: `match_basic_date`/`match_ext_date` match exactly four year bytes (`y1..y4`, each guarded `?0..?9`), so `parse_date`/`parse_utc_datetime`/`parse_naive_datetime` yield years in `-9999..9999`; no oversized integer enters date arithmetic through those parse boundaries.
  - **Conclusion:** the arithmetic is reachable from untrusted input but is not a separate DoS/overflow sink — its cost is dominated and bounded by the already-reported parse limit. Ruled out on executed evidence. **Axes examined:** digit-count scaling (27 → 1.2M digits), day-path (`shift_days`→`date_from_iso_days`), month-path (`shift_months`), parse-boundary year width, over-limit behavior. **Not examined:** none.

- **Huge fractional seconds in `parse_time`/`parse_naive_datetime`/`parse_utc_datetime`** — `parse_microsecond/3` walks trailing digits tail-recursively but converts only the first ≤6 via `:binary.part(digits, 0, precision)`; linear and safe. Basis: verified (`parse_time("00:00:00." <> 5_000_000×"9")` → `{:ok, …}` in ~9 ms). **Axes examined:** digit-count scaling, precision truncation. **Not examined:** none.

- **Atom exhaustion in `parse_duration`** — returned keyword keys (`:year`,`:month`,`:week`,`:day`,`:hour`,`:minute`,`:second`,`:microsecond`) are fixed literals chosen by `find_date_unit/2`/`find_time_unit/2` (`iso.ex:767-776`); no input byte becomes an atom. Basis: verified by reading both helpers. **Axes examined:** key derivation source (literal vs input-derived). **Not examined:** none.

- **`zero_pad/2` infinite recursion / `scale_factor` clause crash** — the negative-value clause recurses only when `count > 6`; every in-file call site passes a literal count ≤ 6 or a `precision` constrained to 1..6 by `scale_factor`'s domain and the `is_microsecond` guard. Runaway branch unreachable via any public path. Basis: reasoned over all in-file call sites (whole file read). **Axes examined:** call-site count values. **Not examined:** none (no dynamic count reaches this).

- **Division-by-zero in `time_from_day_fraction/1`, `add_day_fraction_to_iso_days/3`** — reachable only by a `Calendar.day_fraction()` with a zero denominator, which violates the documented type contract; parser paths always use `@parts_per_day`. Basis: reasoned; excluded per scope (contract violation — these are `@doc false`/`@impl` internals whose input is caller-constructed). **Axes examined:** parser-supplied denominator source. **Not examined:** direct hand-built zero-denominator `iso_days` (contract-violating, out of scope).

- **Offset arithmetic in `parse_utc_datetime`** — `parse_offset/6` caps hour < 24 and minute < 60, so the shift applied by `add_time_unit_to_iso_days/3` is bounded; date validity is checked before the arithmetic branch (`iso.ex:658-681`). No crash/overflow. Basis: verified (`"2015-01-23 23:50:07+02:30"` → `21:20:07`, offset 9000, per doctest). **Axes examined:** offset magnitude bound, validity-before-arithmetic ordering. **Not examined:** none.

## Did I verify this
- **Verification harness.** All behavioral claims were run this pass with the sandbox `elixir` (1.21.0-dev, Erlang/OTP 29, erts-17.0.2). Reproduced examples match this source's documented outputs, but the running standard library may not be byte-identical to the file under review; the `SystemLimitError` threshold depends on the OTP integer-size limit and will differ across ERTS versions (the *presence* of an uncaught raise for a sufficiently large component does not).
- **Cross-boundary flow executed, not assumed.** The previously hedged "outside this file" dismissal was replaced by actually running `Duration.from_iso8601/1` → `Date.shift/2` into this file's `shift_date`/`shift_days`/`shift_months`. The bignum arithmetic is fast (≤1 ms) and non-crashing across day- and month-paths at up to 1.2M-digit components; it is not an independent sink.
- **Severity is contained-by-context.** The one finding is a supervised-process crash / error-handling bypass, not memory unsafety or code execution. Real-world weight depends on whether callers cap input size before `Duration.from_iso8601/1`; below ~1.3 MB it is not reachable.
- **Out of scope, not re-audited:** paths reachable only via `@doc false` internals (`from_unix/2`, `date_to_iso_days/3`, `add_time_unit_to_iso_days/3`, `microseconds_to_iodata/2`) called with contract-violating arguments, and downstream range validation performed by `Duration`/`Date`/`DateTime` callers (their *acceptance* of huge shifted years is their contract, not this file's).
- **Not applicable to this file:** the standing Unicode/identifier-security obligations (UAX #31/#9, UTS #39/#55, bidi/NFC/mixed-script) target the compiler front-end in `lib/elixir/src`; `calendar/iso.ex` performs byte-level ASCII digit/separator matching only and implements none of those code paths, so there is nothing here to satisfy or bypass.

### Dropped in re-review
None. Every prior ruled-out entry and the single finding are preserved. The only change is that the **Bignum blow-up** entry's hedge ("a contract concern outside this file's untrusted-input surface") was removed and replaced with executed evidence: the untrusted flow *does* reach the in-file arithmetic, but the arithmetic is bounded and non-crashing, so the entry remains ruled out — now with `Not examined: none`.
