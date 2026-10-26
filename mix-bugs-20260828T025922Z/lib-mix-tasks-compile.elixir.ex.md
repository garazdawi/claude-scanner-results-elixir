# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.elixir.ex
<!-- sha256: d5e29d8ec2adaaef745a79f76cbd6b5d8c3bc1d052ebcf4bf9f8da522112ba7e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:42:05Z_

## Summary

`compile.elixir.ex` is a thin `Mix.Task.Compiler` wrapper that parses CLI switches/project config and delegates to `Mix.Compilers.Elixir.compile/7`; it is largely clean, with one behavioral asymmetry in how the `:profile` option is normalized that silently discards a valid project-config value.

## Bugfixes

#### `CSCAN-compile.elixir-profile-atom-dropped`
- **Finding ID:** `CSCAN-compile.elixir-profile-atom-dropped`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `compile.elixir.ex:267-273`, `profile_opts/1` (reached from `run/1` at line 153)
- **Class:** Missing clause / non-total function (asymmetric normalization; also documentation/contract mismatch)
- **Impact:** A user who enables compile-time profiling through the documented `:elixirc_options` project config — e.g. `elixirc_options: [profile: :time]` — has that option silently deleted before it reaches the compiler, so no timing information is ever emitted. The equivalent CLI form `--profile time` works. The feature silently does nothing when configured the natural (atom) way in `mix.exs`, with no warning or error to indicate why.
- **Evidence:** `verified`. The `:profile` value flows from `project[:elixirc_options]` into `base` (line 141), survives the merge into `opts` (line 151), and then hits `profile_opts/1`. The function only preserves the string `"time"`; any other `{:ok, _}` — including the atom `:time` that a config author would naturally write — falls to the second clause and is deleted. Reproduced in isolation:
  ```
  $ elixir pt.exs
  cli string time: [profile: :time]
  config atom :time: []      # <-- valid config value silently dropped
  cli string foo: []
  none: []
  ```
  The docs (lines 84, 94-97) state `:elixirc_options` "supports many of the options above" including `--profile`, so setting `profile: :time` there is a documented-public-API path, yet it is discarded.
- **Suggested fix:** Accept the already-normalized atom as well, e.g. add a clause `{:ok, :time} -> opts` before the catch-all, or normalize both forms: `{:ok, v} when v in ["time", :time] -> Keyword.put(opts, :profile, :time)`.

## Risks / caveats

- The file delegates almost all real work to `Mix.Compilers.Elixir`, `Mix.Project`, and `OptionParser`, which are out of scope; I reviewed only the argument/config marshalling in this file.
- I could not exercise the full `run/1` path end-to-end (it requires a loaded Mix project and build lock), so the `cache_key` construction (line 146-147) — which captures `"--no-debug-info" in args` but not the positive `--debug-info` flag that the docs at line 66-67 say "will force a full recompilation" — is a *possible* discrepancy I could not confirm; whether `--debug-info` forces a rebuild may be handled inside `Mix.Compilers.Elixir` via the merged `opts`, which I did not audit. Left unreported as it is not verifiable within this file.
- The `opts ++ Keyword.take(project, [:consolidate_protocols])` construction (line 161) can produce duplicate `:consolidate_protocols` keys, but keyword-list first-match semantics make the command-line value win as intended, so it is not a defect.

### Ruled out
- **Duplicate `:consolidate_protocols` key (line 161)** — verified by reasoning about `Keyword` first-match semantics: the CLI-set value precedes the appended project value, so `Keyword.get` downstream returns the CLI value, matching intent. No wrong behavior.
- **`project[:xref][:exclude]` on missing `:xref` (line 238)** — `nil[:exclude]` returns `nil` via `Access`, and `List.wrap(nil)` is `[]`, so the deprecation branch is correctly skipped; no crash.
- **`is_list(srcs) and Enum.all?(...)` (line 133)** — both operands are genuine booleans, so `and` is used correctly; raising on a malformed `:elixirc_paths` is the intended contract.
- **`pop_tracers`/`Module.concat([str])` (line 258)** — `Module.concat` with a single-string list yields the expected `Elixir.<Name>` atom; multiple `--tracer` values are collected via `:keep`/`pop_values` correctly.

