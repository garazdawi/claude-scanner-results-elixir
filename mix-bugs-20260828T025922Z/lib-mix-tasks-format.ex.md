# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/format.ex
<!-- sha256: a47b2164c377912edb0efe89b943f79e72b503d3f598a14c04241c9d3231376b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:57:55Z_

## Summary

`/Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/format.ex` is the stock `mix format` task (formatter config resolution, plugin loading, file formatting, and the `--check-formatted` text-diff renderer). I reviewed the documented public entry points (`run/1`, `formatter_for_file/2`, `formatter_opts_for_file/2`, and the diff renderer reachable through them) and exercised the diff/format paths at runtime; I found no reachable defect that a well-behaved caller using the documented API can trigger.

## Bugfixes

No findings.

## Risks / caveats

- I verified the diff renderer (`text_diff_format/3` → `diff_to_iodata` and friends) at runtime against mid-file changes, CRLF/`\r` handling, trailing-newline-only differences, and empty-vs-content, checking line-number gutters and skip markers; all produced correct output with no crashes (see the two `elixir` runs above).
- The formatter-config resolution paths (`eval_deps_and_subdirectories`, `eval_subs_opts`, `recur_formatter_opts_for_file`, `expand_dot_inputs`) depend heavily on `Mix.Project`, manifest files, and filesystem layout that I reasoned about but did not stand up a full multi-subdirectory project to drive; the control flow and return shapes are internally consistent.
- This appears to be unmodified upstream Elixir source; the behaviors below were considered and rejected as in-scope bugs.

### Ruled out

- **`format_file/2`'s `rescue` does not catch `throw`/`exit`** (line 800) — an error-handling asymmetry: if a work function `throw`s or `exit`s instead of raising, `Task.async_stream` yields `{:exit, reason}`, which `collect_status/2` has no clause for, causing a `FunctionClauseError` in the reducer. But the only functions in the format path that can fail are `Code.format_string!`, `File.read!`, and `IO.iodata_to_binary`, all of which *raise* (and are caught). The sole way to reach `throw`/`exit` is a user plugin whose `format/2` throws/exits, which violates the `@callback format(String.t(), keyword) :: String.t()` contract — out of scope per the contract-violation rule.
- **`maybe_cache_timestamps/3` touches the timestamp in an `after` even when `fun` raises** (lines 312–317) — I checked reachability: the timestamp-caching branch runs only when `all_args == []`, and `--check-formatted`/`--dry-run`/`--force` all appear in `all_args`, so a *failed check* never reaches this branch. The remaining case (a plain `mix format` where one file raises a real formatting error) records the timestamp after formatting the good files; this is the intended "record progress" behavior of the stock task, not a defect.
- **`colorize/4` space-run guard only tests `["\t", "\s"]`** (lines 1058–1078) — a whitespace run beginning with `\r`/form-feed would be colorized as text rather than background. Since lines are pre-split on `\n` and `\r` is rewritten to the `↵` symbol upstream via `insert_cr_symbols`, this is at most a cosmetic color difference, not a behavioral bug, and does not crash (the empty-string boundary segments fall through to the text clause cleanly — verified by the diff runs producing no error).
- **`insert_cr_symbol/2` replaces *all* `\r`, not just the trailing one** (lines 1119–1125) — only affects the highlight glyph in the diff view when CR presence differs between old/new; purely cosmetic, and the rendered output was correct in the CRLF test above.
- **`recur_formatter_opts_for_file/3` binary prefix match** (lines 743–746) — the `binary-size(^size)` + separator match correctly requires a true path-boundary before descending into a subdirectory's config; no partial-name false match. Return shape `{formatter_opts, root}` is consistent across the recursion.
- **`collect_status`/`check!`/`to_diffs` tuple shapes** — `format_file` emits `{:not_formatted, {file, input, output}}`, which `collect_status` binds whole and `to_diffs` destructures as `{file, unformatted, formatted}` / `{:stdin, …}`; shapes match across the pipeline.

