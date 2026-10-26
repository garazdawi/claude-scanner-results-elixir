# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/formatter.ex
<!-- sha256: a06ef151762a210f272d014db856f9756c4261b0c876440ccd9781b4a87a0ade -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/formatter.ex
<!-- sha256: a06ef151762a210f272d014db856f9756c4261b0c876440ccd9781b4a87a0ade -->

_Re-analysed on 2026-08-27T (re-review pass)_

## Summary

`ExUnit.Formatter` renders test failures, diffs, timings, and filters into human-readable strings for ExUnit formatters. Its inputs are test-result structs supplied by the trusted runner/test code, so the attack surface is intentionally thin. I found one latent robustness/isolation defect in the assertion-diff path (the linked diff `Task`), and no code-exec, atom, deserialization, path, or injection sinks reachable with untrusted data.

In this pass I **executed** the mechanism rather than asserting it: I reproduced the caller-kill and the trap-exit `CaseClauseError` with a genuine `Diff.compute` crash, confirmed the CLIFormatter GenServer does not trap exits (read `cli_formatter.ex`), and fuzzed `Diff.compute` under all three valid contexts to bound the untrusted-data reachability. Net effect: the finding's *mechanism* moves from `asserted` to `verified` (high confidence); its *severity* stays **Low**, because the only crashing inputs I found are carried by the developer-supplied `context` field, not by untrusted runtime data.

## Security

#### `CSCAN-formatter-find-diff-linked-task-crash`
- **Finding ID** — `CSCAN-formatter-find-diff-linked-task-crash`
- **Severity** — Low
- **Confidence** — high (mechanism executed end-to-end; severity bounded by executed fuzzing)
- **Location** — `formatter.ex:558-565`, `find_diff/2` (reached from public `format_assertion_diff/4` → `format_sides/6` → `format_diff/4`)
- **Issue** — `find_diff/2` computes the diff in a **linked** `Task.async(Diff, :compute, [left, right, context])` and then does `case Task.yield(task, 1500) || Task.shutdown(task, :brutal_kill) do {:ok, diff} -> …; nil -> … end`. If `Diff.compute` terminates abnormally: (a) the link established by `Task.async` propagates the exit signal to the caller, killing it **uncatchably** (a surrounding `try/rescue/catch` cannot intercept a link exit); and (b) if the caller happens to trap exits, `Task.yield` returns `{:exit, reason}`, which matches neither `case` branch and raises `CaseClauseError`. The 1500 ms timeout shows the authors meant to contain `Diff.compute` misbehaviour, but the containment only covers *slowness*, not *crashes*.
- **Class** — Concurrency (linked Task lifetime + unhandled abnormal-exit outcome)
- **Impact** — The formatting functions run inside the `ExUnit.CLIFormatter` GenServer's `handle_cast` (verified: `cli_formatter.ex:7` `use GenServer`, `init/1` at `:14-48` returns `{:ok, config}` with **no** `Process.flag(:trap_exit, true)`). A single diff-task crash therefore kills the shared formatter process rather than being contained to one failing assertion, disrupting reporting for the rest of the run — the "one bad test corrupts the run of its siblings" outcome the isolation model exists to prevent. Because the kill is a link exit signal, the formatter cannot defend itself with `try`.
- **Evidence** — `verified` (mechanism, executed). Concrete crashing input, contract-valid struct: `%ExUnit.AssertionError{left: quote(do: ^p), right: 1, context: {:match, []}}` — a normally-shaped `{:match, pins}` context whose `pins` map simply does not contain the pinned variable `p`. `Diff.compute` reaches `diff_pin/3` (`diff.ex:313`) which does `%{^identifier => pin_value} = pins` and raises an **uncaught** `MatchError` (`catch :undiffable` at `diff.ex:41-43` does not cover it; same uncaught crash via `literal_key/2` at `diff.ex:721` for a `%{^p => x}` key). Running the exact `find_diff/2` body against this input:
  - **(A)** a plain caller wrapped in `try/rescue/catch` was **killed uncatchably** — `CALLER KILLED, reason: {{:badmatch, %{}}, [{ExUnit.Diff, :diff_pin, 3, …}]}`.
  - **(B)** a `trap_exit` caller instead observed `{:rescued, CaseClauseError}`, confirming the `{:exit, _}` branch gap.
  A second contract-valid trigger is any `context` atom other than `:==`/`:===` (e.g. `context: :foo`): `context_to_env/1` (`diff.ex:45-49`) has no matching clause → uncaught `FunctionClauseError` → same task crash. `context` is typed `any` in `ExUnit.AssertionError.t()`, so both are contract-valid inputs to the documented `format_assertion_diff/4`.
- **Reachability via genuinely untrusted data (the severity question)** — I tried the axes rather than assuming. Fuzzed `Diff.compute`:
  - `:==` and `:===` contexts — 1682 hostile `left`×`right` pairs (invalid UTF-8, improper lists, structs vs maps, bitstrings, pids/refs/functions, deep nesting, huge ints, escaped-meta AST): **0 crashes**.
  - `{:match, pins}` with **well-formed** pins over 2400 pattern×right×pin combos: the only crashes were the missing-pin `MatchError` above — i.e. driven by the `pins` map (part of the developer-supplied `context`), never by the untrusted `right` value alone.
  So the "hostile runtime value under a valid, well-formed context crashes the diff" vector remains **not demonstrated**; the crashing element in every reproduction is the developer/macro-supplied `context`, which the real `assert`/`assert_receive` macros always emit consistently. Hence severity stays Low, not Medium.
- **Suggested fix** — Make the diff computation degrade to "no diff" on any abnormal termination: use `Task.Supervisor.async_nolink/3` (or `spawn_monitor`) so no link exists, and handle every outcome, e.g. add `{:exit, _reason} -> nil` to the `case`. That keeps the existing 1500 ms cost bound while turning a diff crash into a missing diff rather than a dead formatter.

### Ruled out
- **`format_filters/2` `CaseClauseError` on an atom other than `:include`/`:exclude`** (`formatter.ex:249`) — `verified`. The sole caller is `CLIFormatter.print_filters/2`, invoked only as `print_filters(opts, :exclude)` / `print_filters(opts, :include)` (`cli_formatter.ex:27`), which imports and calls `format_filters/2` with exactly those two atoms. A plain formatting crash with no untrusted-data path and no security impact. *Axes examined: caller set (only CLIFormatter), atom domain (`:include`/`:exclude`). Not examined: none.*
- **`get_code/2` reads a source line from a stacktrace-derived path** (`formatter.ex:577-592`) — `verified` by reading. `file`/`line` come from the runtime-generated stacktrace (compiled-in source paths); the whole body is `rescue`-wrapped (`:590-592`), only one trimmed line is streamed, and the test process already has full filesystem access. No privilege crossing, no untrusted-data path through the public API. *Axes examined: input origin (stacktrace info map), error containment (rescue), FS-privilege delta. Not examined: none.*
- **`with_location/1` `Path.relative_to_cwd`/`Path.join`** (`formatter.ex:712-720`) — `verified` by reading. Inputs are the test's own `:file` tag and the operator-set `:test_location_relative_path` app-env; the result is a display string only, with no subsequent file operation. *Axes examined: both input origins, downstream use (display-only). Not examined: none.*
- **Terminal-escape passthrough in raw-emitted strings** (`format_message/2`, `format_test_failure` description/params, exception banners) — `verified` (design-level). `inspect`-rendered values are escaped; the raw-emitted paths carry test descriptions, parameters, and exception messages authored by the trusted test writer. ExUnit (like `Logger`/`inspect`/`IEx`) does not sanitise ANSI anywhere in error text; this is systemic accepted behaviour requiring a trusted author to route untrusted bytes into a message. *Axes examined: which strings are raw vs escaped, authorship of each. Not examined: none — treated as accepted systemic behaviour, not a formatter defect.*
- **`inspect`/diff resource consumption** — `verified`. `inspect`/`inspect_multiline` run with bounded width/limit, and `Diff.compute` cost is bounded by the 1500 ms task timeout + `brutal_kill`. (Prior pass measured a 1 MB invalid-UTF-8 diff returning in ~0.165 s with the caller surviving the shutdown.) *Axes examined: time bound (task timeout), shutdown path. Not examined: none for a crash/DoS-via-time angle; unbounded-memory-within-1.5 s is theoretically possible but the timeout+kill caps it.*

## Risks / caveats
- Severity hinges on whether a genuinely untrusted runtime value (the `assert expected == untrusted` path) can crash `Diff.compute` under a *well-formed* context. Executed fuzzing (1682 `==`/`===` pairs + 2400 match combos) found no such crash — every crash I could produce is carried by the developer-supplied `context`/`pins`, not by `right`. A future input that crashes `Diff.compute` from `right` alone under a well-formed context would upgrade this to Medium; I did not find one, and I did not exhaustively enumerate every `{:match, pins}` guard/AST shape (guard evaluation via `Code.eval_quoted` at `diff.ex:300` was probed but not exhausted).
- `context: any` in `ExUnit.AssertionError.t()` is what makes both demonstrated triggers contract-valid. A reviewer reading the *implicit* contract as "context ∈ {`:==`,`:===`,`{:match, well-formed-pins}`,`{:mailbox,_}`}" would treat the trigger as out-of-contract and lean toward ruling the finding out; I keep it Low rather than dropping it because the field's declared type admits the input and the crash is real and uncatchable.
- `@doc false` helpers (`format_assertion_error/1`) and deprecated `format_time/2` / `format_test_case_failure/5` only delegate to analysed functions and add no new sink.

## Did I verify this?
- **Read** `formatter.ex` (full), `diff.ex` (full), `cli_formatter.ex:1-60` — confirmed the CLIFormatter GenServer does **not** trap exits (`init/1` returns `{:ok, config}`, no `Process.flag(:trap_exit, true)`), and that `format_filters/2` is called only with `:include`/`:exclude`.
- **Ran** (Elixir 1.21.0-dev / OTP 29):
  - Fuzz of `ExUnit.Diff.compute` under `:==`/`:===` — 1682 hostile pairs, **0 crashes**.
  - Fuzz under `{:match, pins}` — 2400 combos; crashes only from missing-pin `MatchError`, never from `right` alone.
  - Confirmed the missing-pin `MatchError` (`diff_pin/3`, and `literal_key/2` for `%{^p => x}`) is **uncaught** by `Diff.compute`'s `catch :undiffable`.
  - Reproduced `find_diff/2`'s exact body: non-trapping caller **killed uncatchably** through the link (`{:badmatch, %{}}` in `ExUnit.Diff.diff_pin/3`); trap-exit caller got `CaseClauseError`.
- **Did not run**: an exhaustive enumeration of every guard/AST shape under match context (guard eval via `Code.eval_quoted` probed, not exhausted) — the residual untrusted-data-reachability question that keeps severity at Low.

### Upgraded / changed since first pass
- The main finding's Confidence rises **medium → high** and its mechanism evidence moves **`asserted` → `verified`**: the caller-kill and trap-exit `CaseClauseError` are now reproduced with a real `Diff.compute` crash, and a concrete contract-valid crashing input (`context: {:match, []}` with an uncovered pin) is identified. Not an "upgraded hedged assumption" in the finding-severity sense — severity remains Low — but the first-pass hedge "`Diff.compute` is robust across ~1,700 combinations" is now **qualified**: it is robust under value contexts and well-formed match contexts, but crashes (uncaught `MatchError`/`FunctionClauseError`) on an inconsistent developer-supplied `context`.

### Dropped in re-review
- None. All prior findings and ruled-out entries are preserved.
