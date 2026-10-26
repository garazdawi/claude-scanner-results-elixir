# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/date_range.ex
<!-- sha256: d76c54e451ed96a9876e52db77475226b4d1efc40ffac6e5ca836f5dacaabf81 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:02:50Z_

I have completed a thorough review with reproductions. The file is a pure, well-behaved data-structure implementation. Here is my report.

---

## Summary

`Date.Range` is a pure calendar data-structure module (struct plus `Enumerable`/`Inspect` protocol implementations) with no code-execution, atom-creation, filesystem, network, deserialisation, or injection sinks; every security-relevant edge case reachable through the documented public API (`Date.range/2,3` and the `Enum`/`Inspect` protocols) was tested and behaves safely, so there are **no findings**.

## Security

`No findings`

The Phase-1 sink classes were walked against the whole file (244 lines). None of the dangerous named functions (`Code.eval*`/`compile*`, `String.to_atom`/`Module.concat`/`binary_to_term`, `System.shell`/`os.cmd`, `Regex.compile`, `Path.*`/`File.*`, `:erl_tar`/`:zip`, `EEx.*`, `Base.decode*`, `URI.parse`, `:crypto` comparisons, `:persistent_term`/`:ets`/`Application.put_env`/`Process.put`) appear anywhere in this file. The module performs only integer arithmetic over `first_in_iso_days`/`last_in_iso_days`/`step` and calendar conversions on a caller-chosen calendar. The items considered and rejected are below.

### Ruled out

- **Divide-by-zero via `rem(days - first_days, step)` (`member?`, line 50/53) and `div(last_days - first_days, step)` (`size`, line 188).** A `step` of `0` would raise `ArithmeticError`, but `step` cannot be `0` through the public API: `Date.range/2` forces `±1` and `Date.range/3` guards `is_integer(step) and step != 0` (`date.ex:150`). Reaching `step: 0` requires hand-constructing a `%Date.Range{}`, which violates the `step: pos_integer | neg_integer` type contract — excluded by scope. Basis: **verified** — `Date.range(~D[2000-01-01], ~D[2000-12-31], 0)` raises `ArgumentError "…the step must be a non-zero integer"` before any struct is built.
- **`FunctionClauseError` in private `slice/4` when `remaining == 0` (lines 103–112, no `remaining == 0` clause).** Not reachable: `Enum.slice/2,3` short-circuits empty and out-of-bounds slices before invoking the returned slicing fun. Basis: **verified** — `Enum.slice(range, 3, 0)`, `Enum.slice(range, 100, 5)`, and `Enum.slice(range, 5..2)` all return `[]` without a crash; stepped (`0..10//3`) and descending (`step -1`) slices return correct results.
- **Unbounded resource consumption from `Enum.count`.** `count/1` → `size/1` is O(1) closed-form arithmetic, not iteration. Basis: **verified** — counting a range spanning 365,242,499,270 days (year 1 → year 999,999,999) returns instantly (261 µs for a 3.65M-day range; the near-billion-year range is also instant).
- **Stack overflow while enumerating a large range (`reduce`, lines 146–157).** The `reduce/6` recursion is in tail position (the `fun.(...)` call is evaluated as an argument, the recursive `reduce` is the final expression), so enumeration is O(n) time/memory with no stack growth. Basis: **verified** — `Enum.to_list` of a 73,050-element range completes in 5.6 ms with no crash.
- **`member?` returning a wrong answer (direction/parity/empty handling, lines 34–59).** Modular membership respects step sign and range direction, and empty ranges (`step<0 & first<last`, `step>0 & first>last`) short-circuit to `false`. A `Date` with a non-matching calendar falls to the `{:ok, false}` clause without crashing. Basis: **verified** — ascending step-3, descending step-(-3), empty, and cross-calendar cases all returned the correct boolean.
- **`Inspect` string interpolation (`"…, #{step})"`, line 235).** `step` is always an integer, so `#{step}` cannot inject markup or a rogue `String.Chars`; `first`/`last` go through `inspect/1`, not raw interpolation. No injection sink. Basis: verified by reading — the interpolated value is the integer struct field.
- **`Deserialisation` / the `# TODO: Remove me on v2.0` back-compat clauses (lines 61–78, 94–101, 127–136, 190–197, 217–224, 238–242).** These match `%{__struct__: Date.Range}` maps lacking a `step` key (pre-v1.12 serialised terms) and safely infer `step = ±1`. The `binary_to_term`/`Map` step that would produce such a term happens in the caller, not this file, and the clauses degrade gracefully rather than crash. Not a sink in this file.
- **Behaviour dispatch in `date_from_iso_days/2` (lines 168–173) → `calendar.naive_datetime_from_iso_days/1`.** The calendar module comes from `first.calendar`, fixed at construction by the caller (a documented custom-calendar extension point), not from data crossing a trust boundary inside this file; it is not an externally-influenced module name.

## Risks / caveats

- **Inherent enumeration cost is by design, not a defect here.** Iterating a range (`Enum.to_list`, `Enum.map`, `Enum.reduce`, materialising a large `Enum.slice`) is O(n) in the number of dates, and n is bounded only by the endpoints — which can be billions of days apart since `Calendar.ISO` accepts near-arbitrary years (verified: `Date.new(999_999_999, 1, 1)` succeeds). A caller that builds `Date.range(untrusted_start, untrusted_end)` from network/file input and then *materialises* it could be pushed into large CPU/memory use. This is the same property every `Integer` range has, it is not specific to this module, and this file makes the safe choices where it can (`count` and `member?` are O(1), so bounds/size/membership checks never iterate). Any mitigation belongs at the caller (validate the span before enumerating), not in this file.
- **Coverage.** Findings were judged only against the documented public API (`Date.range/2,3`, the `Enumerable` and `Inspect` protocols). Direct construction of malformed `%Date.Range{}` structs (inconsistent `first`/`first_in_iso_days`, `step: 0`, missing keys) violates the module's type contract and was treated as out of scope per the ground rules; such structs can misbehave but are not reachable through the public API.
- The Unicode/tokenizer obligations in the standing threat model do not apply to this file — it contains no text/identifier handling; that surface lives in `elixir/lib/elixir/src`.

