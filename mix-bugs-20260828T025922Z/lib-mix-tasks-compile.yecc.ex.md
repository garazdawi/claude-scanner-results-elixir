# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.yecc.ex
<!-- sha256: 5ba565d55f25e006ce1661b0b400f542ea7ca6aaffbb0d49a27edcb5b4301207 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.yecc.ex
<!-- sha256: 5ba565d55f25e006ce1661b0b400f542ea7ca6aaffbb0d49a27edcb5b4301207 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (single-file hedges resolved against callgraph neighbours + executed evidence)_

## Summary

`compile.yecc.ex` is a thin `Mix.Task.Compiler` wrapper that drives `:yecc.file/2` through `Mix.Compilers.Erlang.compile/6`. In this re-review I eliminated the two single-file hedges from the first pass:

1. **Return-shape contract** — previously "out of scope for a single-file review." I read the neighbour `Mix.Compilers.Erlang` (`erlang.ex`) and confirmed `do_compile/4` (lines 282–308) matches exactly `{:ok, _, warnings}` and `{:error, errors, warnings}`, i.e. the 3-tuples yecc produces.
2. **Option-override ordering** — previously verified only by *reading* `yecc.erl`. I now **executed** `:yecc.file/2` in the sandbox with user options that conflict with `@forced_opts`, on both a valid and a deliberately-broken grammar, and observed the actual returns. The forced `[report: true, return: true]` (appended after the user options) win, and the callback receives the 3-tuple shape in both the success and error paths.

No genuine, reachable bug found in this file.

## Bugfixes

No findings.

## Risks / caveats

- The task's correctness depends on `Mix.Compilers.Erlang.compile/6` invoking the callback with `(input, output)` where `output` already carries the `.erl` extension and on it matching the callback's 3-tuple returns. Both are now verified against `erlang.ex` (see Ruled out). Bugs *inside* `erlang.ex` (e.g. its manifest read/write, `Task.async_stream` result handling) belong to that file's own review, not this one.
- I did not run a full `mix compile.yecc` against a scaffolded Mix project (that exercises `Mix.Project.config()`, manifest paths, and the recursive compiler machinery, none of which live in this file). The file-local contract that mattered — the yecc option resolution and the callback return shape — was exercised directly against the real `:yecc` and confirmed against the real `erlang.ex`.

### Ruled out

- **Option-override doc mismatch** (`@forced_opts` placed after `yecc_options`) — the moduledoc claims `:report`, `:return_errors`, `:return_warnings` set by the user "have no effect."
  - Confidence: N/A (ruled out, executed).
  - Axes examined: user sets `report:false`+`return:false` on a valid grammar; user sets `return_errors:false`+`return_warnings:false` on an error grammar; forced `[report:true,return:true]` appended after in both cases. | Not examined: none.
  - Evidence (executed in sandbox, parsetools-2.8): with `[{report,false},{return,false}] ++ [{report,true},{return,true}] ++ [{parserfile,"out_ok.erl"}]`, `:yecc.file("test.yrl", …)` returned `{ok,"out_ok.erl",[]}` — the forced `return:true` won (3-tuple, not a bare `ok`). With `[{return_errors,false},{return_warnings,false}] ++ [{report,true},{return,true}]` on a broken grammar, it returned `{error,[{"bad.yrl",[…]}],[]}` — again the forced value won. Last-occurrence-wins confirmed by execution, not just by reading `check_options/options` in yecc.erl. Doc is accurate. Not a bug.
- **Callback return-shape mismatch** — could the callback ever hand `do_compile/4` a shape it doesn't match, causing a `CaseClauseError`/`MatchError`?
  - Confidence: N/A (ruled out, verified by read + execution).
  - Axes examined: success path, error path, both under user opts that try to disable `return`. | Not examined: none for the shapes reachable from this file (the deprecated `{:ok, contents}` / `:error` legacy shapes in `do_compile/4` lines 292–306 are unreachable here because `return:true` is forced, so yecc always emits the 3-tuple — confirmed by AXIS 1/2 execution above).
  - Evidence: `erlang.ex:282–308` `do_compile/4` has clauses for `{:ok, _, warnings}` and `{:error, errors, warnings}`; the executed yecc calls produced exactly those two shapes. Aligned.
- **`parserfile` path correctness** (double/missing extension) — `erlang.ex:245–258` `extract_entries` builds `target = Path.join(dest_dir, module <> "." <> to_string(dest_ext))` with `dest_ext = :erl`, so `output` ends in `.erl`; the callback passes it verbatim as `parserfile`. Execution above wrote `out_ok.erl` from `parserfile: "out_ok.erl"` with no rewriting. No double/missing extension. Not a bug.
- **`opts` keyword collisions** (`parallel`/`force`/`all_warnings`) — `opts = [parallel: true, preload: …] ++ opts` prepends defaults; `OptionParser` (switches `force`/`verbose`/`all_warnings`) produces disjoint keys from `parallel`/`preload`; `erlang.ex` reads each via `opts[:force]`/`opts[:parallel]`/`Keyword.get(opts, :all_warnings, true)`, taking the first (prepended) value where present. No key is both prepended-default and CLI-parsed except none. No misresolution. Not a bug.
- **`is_list(yecc_options)` guard** — non-list `:yecc_options` raises a clear `Mix.raise`; documented validation, not a defect.
- **`preload` badmatch** — `{:ok, _} = Application.ensure_all_started(:parsetools)` badmatches if parsetools fails to start. This is an intentional assertion (the compiler cannot proceed without parsetools) and surfaces as a `MatchError` only in a genuinely broken OTP install; consistent with the sibling leex task. Not treated as a bug.

### Did I verify this?

- Read the target file in full.
- Read neighbour `Mix.Compilers.Erlang` (`erlang.ex`) — confirmed `compile/6` signature/contract (lines 76–98), `do_compile/4` return-shape matching (282–308), and `extract_entries` output-extension construction (245–258). This upgrades the first pass's "out of scope" hedge to a verified read.
- Read sibling `compile.leex.ex` — confirmed byte-for-byte structural mirror (only `:yrl`/`:yecc.file`/`parserfile` vs `:xrl`/`:leex.file`/`scannerfile`).
- **Executed** `:yecc.file/2` (Erlang/parsetools-2.8 in sandbox) with user options conflicting with `@forced_opts`, on valid and broken grammars, observing the real returns. This upgrades the first pass's read-only verification of the option-override ordering to executed evidence, letting the "Option-override doc mismatch" and "Callback return-shape mismatch" entries carry `Not examined: none`.

### Dropped in re-review

None. All prior findings/dismissals are preserved; two ruled-out entries were strengthened from `asserted` (read-only) to `verified` (executed), and no dismissal was upgraded to a finding because the axes that could have produced a bug (user suppressing `return`, error path shape) were tried and returned the safe 3-tuple.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Option-override doc mismatch** (`@forced_opts` placed after `yecc_options`) — I suspected user-supplied `report:`/`return_errors:` in `:yecc_options` might win over the forced values, contradicting the moduledoc. Verified against `/usr/local/lib/erlang/lib/parsetools-2.8/src/yecc.erl`: `check_options/3` reverses the option list and `options/1` resolves each option with `lists:keyfind`, so the *last* occurrence wins. `@forced_opts` is appended after `yecc_options`, so the override works and the doc is accurate. Not a bug.
