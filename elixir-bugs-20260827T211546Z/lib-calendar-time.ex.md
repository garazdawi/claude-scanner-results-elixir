# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/time.ex
<!-- sha256: d67146cf725623177ac6bcf8d13ea5e5dec42a874321a67f198c51b936814c78 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/time.ex
<!-- sha256: d67146cf725623177ac6bcf8d13ea5e5dec42a874321a67f198c51b936814c78 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28 (second re-review — hedge discharge)._

# Time module review (re-review)

- **Summary:** Re-reviewed to discharge the hedge flagged by the first-pass reviewer ("out of scope for this file"). The flagged hedge was the caveat in the prior "Did I verify this" section: *"the one axis I genuinely cannot execute is a calendar whose `time_from_day_fraction`/`time_to_day_fraction` differ numerically from `Calendar.ISO` … so it is out of scope for this file."* That axis **is** tryable in this sandbox, so I tried it rather than asserting it away: I built `CoarseCal`, a calendar that maps a time to a day-fraction on a genuinely different denominator (100 000 parts/day, not delegating to ISO), and pushed values through `compare/2`, `convert/2`, and the general `diff/3` clause. The cross-scale comparisons return correct results, so the axis is now `verified (ran)`, not `out of scope`. All other ruled-out entries were re-executed against the as-built module (sha256 confirmed identical: `/opt/elixir/lib/elixir/lib/calendar/time.ex`, Elixir 1.21.0-dev / Erlang OTP 29). No genuine reachable bugs found. No new findings raised.

## Bugfixes

No findings

## Risks / caveats

- The running `/opt/elixir` build is the exact source under review (`sha256sum` of the running file == the report header `d67146cf…`), so behavioural checks reflect the code being reviewed.
- **Discharged hedge (was "cannot execute a numerically-different calendar; out of scope"):** *Upgraded from hedged assumption in first pass — but resolved by execution, not by promotion to a finding.* I constructed `CoarseCal.time_to_day_fraction/4` returning `{div(secs*100_000, 86_400), 100_000}` (a real alternate scale, no ISO delegation) and ran `compare/2` between `CoarseCal` and `Calendar.ISO` times. Raw fractions genuinely differ in denominator (`{50000,100000}` vs `{43200000000,86400000000}`); the cross-products `parts1*ppd2` / `parts2*ppd1` are `4320000000000000` / `4320000000000000` → `:eq`, and `:lt` / `:gt` are correct on `06:00` vs `18:00` and its swap. The cross-scaling in `compare/2` (`time.ex:756`) and `convert/2` (`time.ex:842-846`) is therefore verified against a differently-scaled calendar, not merely a delegating one.
- **Genuine scope boundary (not a hedge):** the *internal numerical correctness* of `Calendar.ISO`'s own `time_from_day_fraction`/`iso_days_to_unit`/`shift_time_unit` lives in `iso.ex`, not `time.ex`. `time.ex`'s obligation is to destructure the right shapes, pass the right arities/units, and thread precision — all of which are exercised below. A defect *inside* `iso.ex` is that file's responsibility; this is a module-ownership boundary, and I confirmed by execution that `time.ex` consumes the documented callee return shapes (`{:ok,_,time,microsecond}`, `{h,m,s,{us,_}}`) correctly.
- Functions whose `@spec`/guards restrict input (`truncate/2` only `%Time{}`, `from_seconds_after_midnight/3` only `Calendar.microsecond()` tuples with `microsecond in 0..999_999` and `precision in 0..6`, `new/5` integer or `{int,int}` microsecond) raise `FunctionClauseError` on off-contract input. Per the review's scope rule these are not reported.

### Ruled out

All entries re-exercised by execution (`/opt/elixir`, Elixir 1.21.0-dev on Erlang/OTP 29). Cross-calendar clauses reached via a delegating `FakeCal`; the numeric-scale axis additionally reached via a non-delegating `CoarseCal`.

- **`utc_now/2` matches the wrong `from_unix` arity** — `{:ok, _, time, microsecond}` (4-tuple) matches `Calendar.ISO.from_unix/2`. Confidence: high. Basis: verified (ran). Axes examined: all four time units (`:native`→prec 6, `:microsecond`→6, `:millisecond`→3, `:second`→0), ISO and non-ISO (`FakeCal`) calendar targets. | Not examined: none.
- **`diff/3` `:hour`/`:minute` truncation gives wrong sign/magnitude for negatives** — delegates to `:microsecond` diff then `div/2` (truncates toward zero). Confidence: high. Basis: verified (ran). Axes examined: negative crossing (`diff(~T[00:29:11],~T[02:29:10],:hour)` → `-1`), zero magnitude (`:minute` → `0`), positive-truncated (`+1`), negative half-day (`-12`). | Not examined: none.
- **`compare/2` cross-calendar clause mis-scales** — cross-multiplies `parts1*ppd2` vs `parts2*ppd1`. Confidence: high. Basis: verified (ran). Axes examined: same-calendar fast path; delegating cross-calendar (`FakeCal` vs ISO) `:eq`/`:lt`/`:gt`; **numerically-different-scale cross-calendar (`CoarseCal` ppd=100000 vs ISO ppd=86400000000): `:eq` on equal instants (cross-products both `4320000000000000`), `:lt` on `06:00`<`18:00`, `:gt` on the swap.** | Not examined: none. *(This entry's `Not examined` list was "numerically-different calendar / out of scope" in the first pass; it is now cleared by executing `CoarseCal` — upgraded from hedged assumption in first pass, resolved by execution.)*
- **`add/3` precision merge drops or inflates precision** — `max(time_unit_to_precision(unit), precision)`. Confidence: high. Basis: verified (ran). Axes examined: coarser-struct/finer-unit (`21,:millisecond` → `{21000,3}`), `:nanosecond` (stays `{0,6}`), positive-integer unit path (`unit=1000` → `.500000`). | Not examined: none.
- **`add/3` cyclic/negative wrap** — Confidence: high. Basis: verified (ran). Axes examined: `-60`→`22:59:00`, `86400`→identity, `-1 :microsecond`→`23:59:59.999999`. | Not examined: none.
- **`convert/2` loses original precision** — second clause rebinds the microsecond *value* from the target calendar but reuses the source `precision`. Confidence: high. Basis: verified (ran). Axes examined: same-calendar copy; cross-calendar ISO→`FakeCal` preserving prec-1 (`{5,1}`); cross-calendar `FakeCal`→ISO preserving prec-6 (`{40,6}`→`.000040`); cross-scale ISO→`CoarseCal` and `CoarseCal`→ISO (`06:00:00`→`~T[06:00:00]`). | Not examined: none.
- **`to_seconds_after_midnight/1` microsecond field** — returns raw microsecond count untouched. Confidence: high. Basis: verified (ran). Axes examined: sub-second `~N[...23:30:15.999]` → `{84615, 999000}`. | Not examined: none.
- **`shift/2` / `__duration__!` unit validation ordering** — `{:microsecond,{ms,prec}}` case precedes the catch-all rejects. Confidence: high. Basis: verified (ran). Axes examined: valid `microsecond: {100,6}`→`01:00:15.000100`, `Duration` struct path (`second: 65`→`01:16:05`), date-unit reject (`day: 1`→`ArgumentError`). | Not examined: none.
- **`from_iso8601/2` builds struct without calendar then converts** — struct default `Calendar.ISO` routes to the same-calendar `convert/2` clause. Confidence: high. Basis: verified (ran). Axes examined: fractional truncation (`23:50:07.0123456` → `.012345`). | Not examined: none.
- **`to_iso8601/2` non-ISO clause self-call arity** — `… |> convert!(Calendar.ISO) |> to_iso8601(format)` resolves to `/2`, threading `format`. Confidence: high. Basis: verified (ran). Re-verified with an actual non-ISO struct (`%Time{calendar: FakeCal, …, microsecond: {1000,3}}`): `:extended`→`"23:00:13.001"`, `:basic`→`"230013.001"`. Axes examined: non-ISO calendar, both `:extended` and `:basic` formats. | Not examined: none.
- **`diff/3` general (non-ISO) clause** — `to_day_fraction` → `Calendar.ISO.iso_days_to_unit` difference. Confidence: high. Basis: verified (ran). Axes examined: cross-scale `CoarseCal 02:00:00` − `ISO 00:00:00` `:hour` → `1` (the `1` rather than `2` is `CoarseCal`'s own coarse-denominator truncation, computed faithfully by `time.ex` on the fraction the calendar returned — not a `time.ex` defect). | Not examined: none.

### Did I verify this

- Every "verified (ran)" corresponds to an executed command against `/opt/elixir` (`/tmp/rev1.exs`, `/tmp/rev2.exs`), not a read-through of `iso.ex`.
- Delegating cross-calendar clauses (`compare/2`, `convert/2`, `utc_now/2` non-ISO, `to_iso8601/2` non-ISO, `diff/3` general) were reached with `FakeCal`.
- The axis the **first pass declared unexecutable** ("a calendar whose day-fraction differs *numerically* from ISO") was executed this pass with `CoarseCal` (day divided into 100 000 parts, no ISO delegation). `compare/2`'s cross-multiplication produced mathematically correct `:eq`/`:lt`/`:gt` across the differing denominators, and `convert/2` round-tripped a coarse-scale time to ISO. There is now **no** unexecuted axis for `time.ex`'s logic.
- What remains genuinely outside this file: the internal arithmetic of `Calendar.ISO` (and any third-party calendar) — its own module's responsibility. `time.ex` demonstrably passes the right shapes/units/precision to it. This is a module boundary, not an unverified assumption about `time.ex`.

### Dropped in re-review

None. All prior ruled-out entries retained. One additional ruled-out entry (`diff/3` general non-ISO clause) is stated explicitly this pass. No prior finding existed to drop. The sole hedge ("numerically-different calendar / out of scope") was **not** downgraded to a finding because execution showed the code is correct on that axis — it is upgraded from a hedged assumption to `verified (ran)` with cited executed evidence.
