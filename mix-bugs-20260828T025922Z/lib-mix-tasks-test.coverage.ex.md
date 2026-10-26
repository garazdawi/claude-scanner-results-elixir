# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/test.coverage.ex
<!-- sha256: 25d203f21f0ad85ce94de39c8164e1a1fa7486b17274fd129a06703106f86d9f -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/test.coverage.ex
<!-- sha256: 25d203f21f0ad85ce94de39c8164e1a1fa7486b17274fd129a06703106f86d9f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

## Summary

`mix test.coverage` report-generation task; the coverage/analysis logic is total and correct, but the summary-table renderer crashes with an unhelpful `Enum.EmptyError` when the set of non-ignored modules is empty. Two first-pass hedges ("reachability is reasoned, not observed"; ":coverage-level analysis is binary per line ... did not exercise `:cover`") have been resolved by executing `:cover` and the `print_summary` composition in the sandbox.

## Bugfixes

#### `CSCAN-test.coverage-empty-modules-enum-max`
- **Finding ID:** `CSCAN-test.coverage-empty-modules-enum-max`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `elixir/lib/mix/lib/mix/tasks/test.coverage.ex:368`, `print_summary/3` (`is_list(opts)` clause)
- **Class:** Missing clause / non-total function (empty-collection reduce)
- **Impact:** When coverage is generated but there are zero non-ignored modules, `mix test.coverage` aborts with `** (Enum.EmptyError) empty error` from deep inside `Enum.max/1` instead of printing an (empty/100%) summary or a clear message. The caller gets a crash whose stack trace names nothing about coverage or ignore configuration.
- **Evidence:** now **verified** at the function-composition level (was `asserted` in the first pass). Path: `run/1` → `generate_cover_results/1` → `summary(ok, modules, summary_opts)` with `summary_opts` truthy (the default, `Keyword.get(opts, :summary, true)`). `summary/3` calls `gather_coverage(results, keep)` with `keep = modules`; the accumulator `module_results = for module <- keep, do: {...}` is `[]` when `keep == []`, so `print_summary([], totals, opts)` runs:
  ```elixir
  name_max_length = results |> Enum.map(&String.length(elem(&1, 1))) |> Enum.max() |> max(10)
  ```
  with `results == []`. I executed the exact composition in the sandbox (empty `keep` → empty `module_results` → `print_summary` body):
  ```
  module_results for empty keep: []
  CRASH: Enum.EmptyError - empty error
  ```
  So the chain "empty `keep` ⇒ `Enum.max([])` crash at line 368" is observed, not reasoned. `modules` becomes empty when `Enum.reject(:cover.modules(), &ignored?/2)` rejects everything — e.g. an `:ignore_modules` entry (a `%Regex{}` such as `~r//`, or an exhaustive atom list) matching every compiled module, or an app whose `ebin` compiled to zero modules while imported `.coverdata` still satisfies the `[] ->` guard in `run/1`. The `[] ->` guard only covers "no `.coverdata` files found"; it does **not** cover "coverdata present but no reportable modules", so the empty-`keep` path is reachable. What remains code-reading rather than executed is only whether a real project reaches an empty non-ignored-module set — that is config-driven (`:ignore_modules`) and clearly attainable, which is why confidence stays medium rather than high.
- **Suggested fix:** Guard the empty case before computing widths, e.g.:
  ```elixir
  name_max_length =
    results |> Enum.map(&String.length(elem(&1, 1))) |> Enum.max(fn -> 0 end) |> max(10)
  ```
  (`Enum.max/2` supplies a default for `[]`), or early-return with an informative `Mix.shell().info/error` when `results == []`.

## Risks / caveats

- `String.to_unsafe_atom/1` in `beams/1` (line 251) grows the atom table from filesystem-derived beam names; this is by-design per the folder threat model (beams under the build dir are trusted) and is not treated as a finding.
- End-to-end reachability of the empty-modules crash *through a real `mix test.coverage` run* (as opposed to the `gather_coverage`→`print_summary` composition, which I executed) was not driven, because building a full umbrella/`.coverdata` fixture through `run/1` was out of scope for the sandbox. This is the sole residual reason the finding above is medium and not high.

### Ruled out

- **`gather_coverage/2` `case cov do {1, 0} -> …; {0, 1} -> … end` (lines 338–341) is non-total / could raise `CaseClauseError` on a `{0,0}` or `{N,M}` row** — **ruled out, verified by execution.** In the first pass this was an unreported caveat ("I believe `:coverage`-level analysis is binary per line, but did not exercise `:cover`"). I cover-compiled a module in the sandbox (Erlang/OTP 29, `:cover`) and inspected every row of `:cover.analyse(:coverage, :line)` with `line != 0` across all input axes. For `line != 0` the analyser only ever emits `{1,0}` or `{0,1}`; multiple executions still collapse to `{1,0}` (it is boolean-per-line, not a counter), and `line == 0` rows are excluded by the `line != 0` guard before the `case`. The `case` is therefore total on every input this task can feed it.
  - **Axes examined:** covered line (→`{1,0}`); in-module reachable-but-unexecuted line (→`{0,1}`); line executed multiple times (→ still `{1,0}`, not `{2,0}`); never-called module, all lines (→`{0,1}`); export→import→merge path used by `run/1` (`:cover.import` of two coverdata files, then `:cover.analyse` — still only `{1,0}`/`{0,1}`); `line == 0` rows (present as none here, and filtered by the `line != 0` comprehension guard regardless).
  - **Not examined:** none.
- **`color/2` `true`/`false` clauses (lines 399–400) are dead code** — display/3 receives `threshold` from `get_threshold/1`, which returns `@default_threshold` (integer) for `get_threshold(true)` and `Keyword.get(opts, :threshold, @default_threshold)` otherwise. For the ordinary integer/float threshold the `true`/`false` heads never match, so on the numeric path they are unreachable defensive clauses rather than a behavioural bug.
  - **Axes examined:** all `display/3` call sites (per-module rows in `print_summary/3`, and the `Total` row at line 375), each passing the `threshold` produced by `get_threshold/1` on the numeric path.
  - **Not examined:** the config axis where a user sets a non-numeric `:threshold` (e.g. `summary: [threshold: true]`), in which case `get_threshold(opts)` would return that literal and `color(_, true)`/`color(_, false)` *would* match — I did not push a boolean `:threshold` config value through `get_threshold/1` → `color/2`, so I cannot claim the clauses are dead under adversarial config. This does not create a crash (the clauses handle it), so it is not upgraded to a finding.
- **`format_number/2` returning a charlist interpolated via `#{}`** — `to_string/1` on the `:io_lib.format` iolist is handled by `List.Chars`/`IO.chardata_to_string`, producing the correct string; no defect.
  - **Axes examined:** integer input (routed through `format_number(number / 1, length)`), float input (direct `:io_lib.format`), interpolation into a binary at line 381–382.
  - **Not examined:** none material.
- **`html/3` `receive` without timeout (lines 302–306)** — each `pid` from `:cover.async_analyse_to_file` is `Process.monitor`ed and the loop waits on that monitor's `{:DOWN, ^ref, …}`; a monitor is guaranteed to deliver a `:DOWN` on process exit (including if the process is already dead when monitored), so the loop cannot hang.
  - **Axes examined:** normal analyse completion (monitor fires `:DOWN`), analyse process crash (monitor still fires `:DOWN`), ref-matching (`^ref` pins each iteration to its own monitor).
  - **Not examined:** none material.

### Did I verify this?

- **Empty-modules crash (finding):** Verified by execution — reproduced `Enum.EmptyError` by running the `gather_coverage`→`print_summary` composition with empty `keep` in the sandbox Elixir. Reachability of an empty non-ignored-module set is code-reading (`Enum.reject(:cover.modules(), &ignored?)` with an exhaustive `:ignore_modules`), not end-to-end observed; hence medium confidence.
- **`case cov` totality (Ruled out):** Verified by execution — cover-compiled a module and enumerated `:cover.analyse(:coverage, :line)` row shapes across covered / uncovered / multiply-executed / never-called / import-merged inputs; all `line != 0` rows are `{1,0}` or `{0,1}`. `Not examined: none`.
- **`color/2` dead-code (Ruled out):** Verified by reading call sites for the numeric path only; the non-numeric-`:threshold` config axis is explicitly left on `Not examined`, so the "dead code" claim holds only for numeric thresholds.
- **`format_number`, `html/3` receive (Ruled out):** Verified by reading; no execution needed.

### Dropped in re-review

None. Every first-pass finding and ruled-out entry is preserved. The two first-pass hedges ("reachability … reasoned, not observed" and the `:cover` binary-row caveat) were resolved by executing `:cover` (Erlang/OTP 29) and the `print_summary` composition in the sandbox: the crash chain is upgraded from `asserted` to observed, and the `case cov` caveat is upgraded to a fully-examined Ruled-out entry with `Not examined: none`.
