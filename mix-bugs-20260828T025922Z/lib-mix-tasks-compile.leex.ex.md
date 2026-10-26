# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.leex.ex
<!-- sha256: b33cdcb8077d1d31847ffc2673551e1fa5d6d0849bebd8db475260a85cb16d86 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.leex.ex
<!-- sha256: b33cdcb8077d1d31847ffc2673551e1fa5d6d0849bebd8db475260a85cb16d86 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`compile.leex.ex` is a thin `Mix.Task.Compiler` wrapper that drives `:leex.file/2` over `.xrl` sources via `Mix.Compilers.Erlang.compile/6`. It is a structural mirror of `compile.yecc.ex` (the two differ only in the `leex`/`yecc` naming, the `:xrl`→`:erl` vs `:yrl`→`:erl` extensions, and `scannerfile:` vs `parserfile:`). I found no in-scope behavioural defects. Both first-pass hedges ("I assumed …") have been resolved by reading the neighbour modules named in the standing context; neither concealed a defect, and neither required upgrading to a finding.

## Bugfixes

No findings.

## Risks / caveats

- The callback-contract dependency on `Mix.Compilers.Erlang.compile/6` is now verified rather than assumed (see Ruled out, entry 1). I read `compile/6` and `do_compile/4` in full; I still did not audit unrelated parts of that module (manifest read/write, diagnostics formatting), as they are outside this file's contract and belong to `compilers/erlang.ex`'s own review.
- The two shape-critical defect candidates were confirmed in the first pass by running OTP's `leex` built from the source under review (outputs cited below). I did not exercise every leex error/warning permutation, but the `{:ok, _, warnings}` / `{:error, errors, warnings}` contract was confirmed on both the success and multi-option paths, and — critically — `@forced_opts` pins `return: true`, so the callback can never reach `do_compile`'s deprecated 2-tuple/`:error` clauses.

### Ruled out

- **Callback return-shape mismatch with `do_compile`** — *Verified (reading + execution).* `do_compile/4` (`compilers/erlang.ex:282-308`) has four clauses: `{:ok, _, warnings}`, `{:error, errors, warnings}`, `{:ok, _}` (deprecated, `IO.warn`), and `:error` (deprecated, `IO.warn`). The callback here forces `return: true`, and running `leex:file("test.xrl", [{report,true},{return,true},{scannerfile,...}])` returned `{ok,"out.erl",[]}` (a 3-tuple), matching the `{:ok, _, warnings}` clause. With `return: true` forced, leex always yields a 3-tuple/`{:error, _, _}`, so the deprecated clauses are unreachable from this task. The `compile/6` `@spec` (`compilers/erlang.ex:82`) documents exactly `{:ok, term(), [term()]} | {:error, [term()], [term()]}` as the callback's required return — the task satisfies it. **Not examined: none.**
- **Doc claim that `:report`/`:return_errors`/`:return_warnings` in `:leex_options` "has no effect"** — *Verified (execution).* `leex_options ++ @forced_opts` places user options first; to check whether leex's option handling lets the first occurrence win, I ran `[{return_errors,false},{report,true},{return,true}]` → `{ok,"b1.erl",[]}` (return on) vs `[{return_errors,false}]` alone → `{ok,"b2.erl"}` (return off). The forced opts override the user setting regardless of position, so the moduledoc contract holds. **Not examined: none.**
- **`scannerfile` vs `parserfile` option name** — *Verified (execution).* leex's output-file option is `scannerfile` (yecc uses `parserfile`, confirmed at `compile.yecc.ex:67`); this file uses `scannerfile` (`compile.leex.ex:66`). The requested output file `out.erl` was produced at the requested path. **Not examined: none.**
- **`is_list(leex_options)` guard / `Mix.raise`** — *Verified (reading).* `compile.leex.ex:59-61` raises a clear `Mix.Error` on non-list config, identical in shape to `compile.yecc.ex:60-62`. This is the documented `INPUT_INVALID_RETURNS_ERROR`/config-validation contract, not a defect. **Not examined: none.**
- **`preload` starting `:parsetools`** — *Verified (reading).* `preload/1` (`compile.leex.ex:71-81`) is byte-for-byte identical to `compile.yecc.ex:72-82` apart from the `:leex`/`:yecc` atom and message text. `compile/6` invokes `preload` only when `entries != []` (`compilers/erlang.ex:93-95`), i.e. only when there is stale work — so `Mix.ensure_application!(:parsetools)` + `Application.ensure_all_started(:parsetools)` runs before any `:leex.file` call. Correct dependency handling. **Not examined: none.**

### Resolved hedges (from first pass)

- **"I did not audit `compile/6` itself … I only confirmed the callback contract"** — Resolved by reading `compilers/erlang.ex` in full. The callback contract this file must satisfy (`{:ok, term, warnings}` / `{:error, errors, warnings}`) is documented at `compilers/erlang.ex:72-83` and enforced by `do_compile/4:283-308`; the task's forced `return: true` keeps it on the supported 3-tuple path. No defect in this file's use of `compile/6`.
- **"`run/1` correctness depends on runtime state … I assumed that environment rather than reproducing a full Mix project build"** — Resolved by reading the sibling `compile.yecc.ex`. `run/1` here is structurally identical to the known-good yecc task: same `Mix.Project.config()` read, same `assert_valid_erlc_paths`, same `preload` wiring through `compile/6`'s `preload:` option, same `manifest`/`clean` shape. The `Mix.Project.config()` / `:parsetools` runtime dependency is the same well-trodden path the yecc task relies on, not a leex-specific assumption. No defect.

### Did I verify this?

- **Target file read in full:** yes (`compile.leex.ex`, 91 lines).
- **Neighbours read this pass:** `compile.yecc.ex` (structural mirror) and `compilers/erlang.ex` (`compile/6`, `do_compile/4`, `combine_results/2`) — both read in full to convert the first pass's two "I assumed" hedges into reading-verified conclusions.
- **Executed evidence:** the two shape-critical ruled-out entries (return-shape, forced-option override) rest on concrete `leex:file/2` runs whose outputs are cited above; those axes are `Not examined: none`. The reading-only ruled-out entries (guard, preload, option name) concern code that is present and structurally identical to the verified yecc sibling — no untried input axis remains for them.

### Dropped in re-review

None. Every prior ruled-out entry and the "no findings" bugfix verdict are preserved; the two hedges were resolved to verified conclusions rather than dropped or upgraded, because reading the neighbour modules (plus the first pass's executed leex runs) established the claims with no untried disagreement axis remaining.
