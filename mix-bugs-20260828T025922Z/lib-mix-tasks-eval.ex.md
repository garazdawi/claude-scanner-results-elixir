# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/eval.ex
<!-- sha256: 3b4e7327d7f2e1356462b0dc39f4cd9cfc2a23a1c21b756b02c71f486a0b7767 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:54:20Z_

The parsing and control flow behave as the code intends and match upstream `Mix.Tasks.Eval`. This file is a faithful, minimal task wrapper.

## Summary
`mix eval` parses head switches, optionally configures the app (or permits `--no-mix-exs`), then evaluates the given string with a temporarily swapped `System.argv` restored via `try/after`; the logic is correct and matches the documented contract.

## Bugfixes
No findings

## Risks / caveats
- The file is 106 lines with a single public entry point (`run/1`); I read all of it and reasoned through each branch of the `case`/`cond`.
- `Code.eval_string(to_eval)` executes arbitrary code by design (documented behavior mirroring `bin/my_app eval`), so it is out of scope as a "vulnerability."
- I did not exercise the full `Mix.Task.run("app.config", ...)` path inside a real project (would require scaffolding a `mix.exs`), but the arguments passed (`["--no-app-loading" | args]`) match the established upstream contract and the flags in the `@moduledoc`.

### Ruled out
- **`Mix.Task.reenable("eval")` not called when `Code.eval_string/1` raises** — On line 94 `reenable` sits inside the `try` before the `after`; if the evaluated code raises, the task is not re-enabled and the exception propagates. This matches upstream `Mix.Tasks.Eval` behavior and is the intended semantics (a raising eval aborts the mix invocation anyway), so it is not a defect introduced here.
- **`--no-mix-exs` checked against raw `args` rather than parsed `opts`** — `mix_exs: :boolean` is in the strict set, so `--no-mix-exs` is consumed into `opts` as `mix_exs: false`, yet the guard uses `"--no-mix-exs" in args`. Verified (`elixir -e ...`) that `args` still contains the literal `"--no-mix-exs"` string, so the membership test succeeds; this is the intended, upstream-matching idiom, not a bug.
- **Passing the full `args` (including the eval string) to `app.config`** — `app.config` performs its own head/strict option parsing and the code string arrives as a trailing positional it ignores; this is upstream behavior and does not misconfigure the app.
- **`parse_head!` raising on an unknown leading switch** — This is the documented contract of `OptionParser.parse_head!` (bang variant), i.e. crash-on-invalid-input is intended, not a defect.
- **`System.argv` restoration on error** — The `try/after` unconditionally restores `old_argv`, so no argv leak occurs on the eval-raises path.

