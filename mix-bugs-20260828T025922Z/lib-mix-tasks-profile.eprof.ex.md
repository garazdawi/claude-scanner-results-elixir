# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.eprof.ex
<!-- sha256: 0249292b4c9d60f82d66bbf755f9ed6ef10e9d2ed37b16c5b730148ccccf2fe3 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T05:02:32Z_

## Summary

`Mix.Tasks.Profile.Eprof` profiles code with Erlang's `:eprof` and prints a formatted table; the file is largely correct, but the table's column-width computation omits the "Total" row, so that row's cells overflow their columns and the whole table misaligns.

## Bugfixes

#### `CSCAN-profile.eprof-total-row-excluded-from-widths`
- **Finding ID:** `CSCAN-profile.eprof-total-row-excluded-from-widths`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/mix/lib/mix/tasks/profile.eprof.ex:279`, `print_result/1`
- **Class:** Off-by-one / wrong arithmetic (column-width computation excludes a printed row)
- **Impact:** The profiler's output table is misaligned. `column_lengths/2` is fed only the header and the per-function rows (`column_lengths(@header, formatted_rows)`), but `format_total/2` is printed with those same widths. The Total row's hard-coded `"100.00"` percentage is 6 characters, whereas with two or more matching functions no single function reaches 100 % so the widest per-function `%` cell is 5 characters (`"27.69"`). The `%` column is therefore always one character too narrow for the Total row, and its cell overflows the field, shifting every subsequent column of the Total row out of alignment with the header and the data rows. The same happens for `CALLS`/`TIME` whenever the summed totals have more digits than any single function row. It is a readability defect only — the numbers themselves are correct.
- **Evidence:** `verified`. I recompiled the file under review and ran the real public API `Mix.Tasks.Profile.Eprof.profile/2`, then compared it against a copy patched to `column_lengths(@header, [formatted_total | formatted_rows])`:

  Buggy (current source):
  ```
  #                                                CALLS     %  TIME M-5S/CALL
  Total                                           700004 100.0 41142    0.06
  Enum.reduce_range/5                             100001 20.25  8330    0.08
  ```
  Fixed (Total included in widths):
  ```
  #                                                CALLS      %  TIME M-5S/CALL
  Total                                           700004 100.00 40674    0.06
  Enum.reduce_range/5                             100001  20.24  8234    0.08
  ```
  In the buggy output the `%` column is 5 wide so the Total's `100.00` overflows and the `%`/`TIME` columns no longer line up with the header and data rows; in the fixed output the `%` column widens to 6 and everything aligns. The sibling task `profile.tprof.ex:391`, written from the same template, correctly uses `column_lengths(header, [formatted_total | formatted_rows])`, confirming the intended behavior. (The garbled `M-5S/CALL` / `µ` is just this sandbox's latin1 terminal encoding, unrelated to the defect.)
- **Suggested fix:** Include the total row when computing widths, mirroring `profile.tprof.ex`:
  ```elixir
  column_lengths = column_lengths(@header, [formatted_total | formatted_rows])
  ```

## Risks / caveats

- The environment's `elixir`/`mix` at `/opt/elixir` is 1.21.0-dev built from Erlang/OTP 29. To avoid depending on whatever was pre-compiled there, I recompiled the exact source file under review with `ignore_module_conflict` and drove the real `profile/2` public entry point, so the reproduction reflects this file's code.
- I reviewed the remaining paths (`run/1` argument threading into `Mix.Tasks.Run.run/5`, `parse_opt/1`, `filter_results/2`, `sort_results/2`, `add_totals/1`, `divide/2` zero-guard, `format_row`/`format_total`) and found no additional behavioral defects; `divide/2` correctly guards against division by zero for the empty/zero-time cases, and `Run.run/5` is called with the documented `(args, opts, head, expr_evaluator, file_evaluator)` shape.

### Ruled out
- **`:eprof.profile([], fun, matching, set_on_spawn: …)` option validity** — I worried `set_on_spawn:` might be an unrecognized eprof option; the recompiled real run executed the profile without error, so the option is accepted. Not a bug.
- **`opts` passed to `Mix.Tasks.Run.run` before `parse_opt/1` mapping** — `Run.run/5` only consumes `require`/`eval`/`parallel`/`config`, and `profile_code/2` applies `parse_opt/1` before building the profiled expression, so the unmapped `matching`/`sort` keys never reach code that would misinterpret them. Verified the run path executes end-to-end.
- **`divide/2` producing a wrong percentage on zero total time** — the `divide(_, 0) -> 0.0` clause handles it; the empty-results path routes through `print_output([])`. No crash or NaN. Confirmed by running with a fast, near-zero-time function.

