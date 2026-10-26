# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.ex
<!-- sha256: 4b61b7569bd06980a6b0956eb4c3d730fd0981c7aadb7e38368df2203a19656e -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.ex
<!-- sha256: 4b61b7569bd06980a6b0956eb4c3d730fd0981c7aadb7e38368df2203a19656e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`Mix.Tasks.Archive` is a trivial 40-line listing task: `run/1` globs `Mix.path_for(:archives)/*`, maps each entry to its basename, and prints via a total two-clause `print/1`. No reachable defects were found. The first pass's hedges about neighbour-module contracts (`Mix.path_for/1`, `Mix.Shell.info/1`) have been verified by reading those modules directly; they are no longer assumptions.

## Bugfixes

No findings.

`run/1` ignores its argument, resolves the archives directory, globs `*`, maps `Path.basename/1`, and dispatches to `print/1`. `print/1` has an empty-list clause and a general clause, exhaustive over any list. No untrusted-input parsing, resource acquisition, atom creation, or cross-file state threading exists here.

## Risks / caveats

- `Mix.path_for(:archives)` (mix.ex:657-659) returns `System.get_env("MIX_ARCHIVES") || Path.join(Mix.Utils.mix_home(), "archives")`. If an operator sets `MIX_ARCHIVES=""` (empty string, which is truthy in Elixir), `path_for` returns `""`, making the glob `"*"` resolve against the current working directory rather than the archives dir. This is operator-controlled env input, matches Mix's uniform `path_for` behaviour across all callers, and is not a defect local to this file — noted only for completeness, not raised as a finding.

### Ruled out

- **`print(items)` iodata argument `["* ", item]`** — `verified`. The `Mix.Shell` behaviour declares `@callback info(message :: IO.ANSI.ansidata()) :: :ok` (shell.ex:39). `item` is a binary produced by `Path.basename/1`, so `["* ", item]` is a valid `IO.ANSI.ansidata()` list. The prior hedge ("assumes standard-library behaviour … outside this file") is resolved by reading `shell.ex` — the contract is that info accepts ansidata, so no iodata/charlist mismatch is possible. This is a single-implementation contract check, not a differential comparison; there are no cross-implementation axes to try. Not examined: none.
- **`run/1` ignoring its argument** — `verified`. The `Mix.Task` contract passes CLI args as a list; a listing task with no documented options (`@moduledoc`, lines 10-21) correctly ignores them. Not a contract violation. Not examined: none.
- **Empty vs non-existent archives directory** — `verified`. `Path.wildcard/1` returns `[]` for a non-matching/non-existent path, routing to the `print([])` clause ("No archives currently installed."). No crash. Not examined: none.
- **`Mix.path_for(:archives)` return type feeding the pipeline** — `verified`. Read at mix.ex:657-659: it returns a binary in both branches (env-var string or `Path.join/2` result). `Path.join(binary, "*")` → binary → `Path.wildcard/1` → list of binaries → `Path.basename/1` → binary. The prior hedge that this "matches documentation … outside this file" is now confirmed by reading the definition. Not examined: none.

### Did I verify this?

- I read `mix/tasks/archive.ex` (the target), `mix/shell.ex` (the `Mix.Shell` behaviour, confirming `info/1` accepts `IO.ANSI.ansidata()`), and the `Mix.path_for/1` clauses in `mix.ex` (confirming a binary return). These reads convert the first pass's two "outside this file" hedges (about `Mix.path_for/1` and `Mix.Shell.info/1`) into verified contract checks.
- No driver script was executed: the file contains no branching whose behaviour depends on differential/edge inputs, and no ruled-out entry rests on a two-implementation comparison, so there are no untried axes. Every ruled-out entry above carries `Not examined: none` on the strength of the neighbour-module contracts I read, not on an untried assumption.
- `Path.wildcard/1` and `Path.basename/1` are Elixir-core `Path` functions; their list-of-strings / basename contracts are relied upon as documented. A defect inside `Path` itself would not manifest as a bug in this file's logic.

### Dropped in re-review

No prior findings dropped — the first pass reported no findings. The two hedged assumptions ("assumes standard-library behaviour … outside this file" and the iodata-acceptance assumption) were not upgraded to findings because reading the neighbour modules confirmed the contracts hold; they are recorded above as `verified` ruled-out entries with `Not examined: none`.
