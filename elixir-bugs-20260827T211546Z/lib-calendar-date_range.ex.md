# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/date_range.ex
<!-- sha256: d76c54e451ed96a9876e52db77475226b4d1efc40ffac6e5ca836f5dacaabf81 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:40:14Z_

I've reviewed the file thoroughly and exercised its public surface (`Enum.member?`, `count`, `slice`, `reduce`/`to_list`, stepped/negative/empty ranges, and `Inspect`) against the built compiler.

## Summary
`Date.Range`'s `Enumerable` and `Inspect` implementations are correct and total across ascending, descending, stepped, empty, and boundary cases; I found no genuine reachable bugs.

## Bugfixes

No findings

## Risks / caveats

- I could not exercise the non-ISO-calendar branch of `date_from_iso_days/2` (lines 168–173) with a real alternative `Calendar` implementation, since none ships in core. Its logic (delegating to `calendar.naive_datetime_from_iso_days({days, {0, 86_400_000_000}})` at midnight) is straightforward and mirrors the ISO branch, so I judged it correct by inspection, but this is `asserted`, not `verified`.
- The three `# TODO: Remove me on v2.0` clauses (member?/slice/reduce/size/empty?/inspect deprecated heads) exist to handle pre-v1.12 structs lacking `:step`. These are not reachable through the documented public creation path (`Date.range/2,3` always populate `:step`); they'd only fire on a deserialized legacy term. I treated them as out of scope per the "documented public API only" rule.

### Ruled out
- **`slice/4` (lines 103–112) crashing on a zero-length slice (no `remaining == 0` clause).** Enum clamps a zero-length request before invoking the slicing fun, so the private `slice/4` is never called with `remaining == 0`. Basis: `verified` — `Enum.slice(r, 0, 0)` and `Enum.slice(empty_range, 0, 3)` both returned `[]` with no `FunctionClauseError`.
- **`member?/2` first clause (lines 34–55) mishandling the `rem` alignment check for negative steps / descending ranges.** `rem(days - first_days, step)` yields 0 exactly on aligned dates in both directions. Basis: `verified` — membership tested true/false at aligned/unaligned dates on both an ascending step-2 range and a descending step-2 range (`member 03`→true, `member 02`→false, `member 08`→true).
- **`member?/2` returning `{:ok, false}` for a `Date` in a different calendar (second clause, lines 57–59).** This is consistent with the enumeration (which only emits same-calendar dates), so it is intended behavior, not a defect. Basis: `asserted` (reasoned from the enumeration semantics; no alternate core calendar available to drive it).
- **`size/1` off-by-one when `(last-first)` is not divisible by `step` (line 188).** `div/2` floors toward the correct inclusive count. Basis: `verified` — `Enum.count` matched `length(Enum.to_list(...))` for step-2 (5), step-3, descending, and empty ranges.
- **Empty-range handling across `count`/`slice`/`member?`/`reduce`.** All return `0`/`[]`/`false`/`[]` respectively. Basis: `verified` — `Date.range(~D[2020-01-10], ~D[2020-01-01], 2)` yielded empty count 0, empty slice `[]`, empty member `false`, empty list `[]`.
- **`Inspect` picking the wrong arity form (lines 230–236).** Step `1` prints the two-arg form; other steps (incl. `-1`) print the three-arg form. Basis: `verified` — inspected step-1, step-3, and default-descending (`-1`) ranges produced `Date.range(a, b)`, `Date.range(a, b, 3)`, and `Date.range(a, b, -1)`.
- **Stepped/`Enum.slice` with a range step (arity-3 slicing fun, line 91).** Composes range step and Enum step correctly. Basis: `verified` — `Enum.slice(r, 0..9//3)` returned every-third date, and `Enum.at`/`take_every` were correct.

