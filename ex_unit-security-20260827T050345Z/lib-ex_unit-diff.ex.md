# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/diff.ex
<!-- sha256: 77b88df90d6d8b3a497bdefb8fad18f40c5b0df24f260cddc000617bfa700332 -->

_Analysed on 2026-08-27T05:31:36Z_

## Summary

`ExUnit.Diff` computes and renders the coloured left/right diff shown when an assertion fails; it cleanly separates *AST* (the developer-written pattern/expression, which it may `Code.eval_quoted`) from *values* (the runtime data under comparison, which it only escapes/inspects and never evaluates), so its two code-execution sinks are not reachable from value data — the one real defect is an exponential-time blow-up in the list-diff path.

## Security

#### `CSCAN-diff-nested-list-exponential-rediff`
- **Finding ID** — `CSCAN-diff-nested-list-exponential-rediff`
- **Severity** — Low
- **Confidence** — high
- **Location** — `diff.ex:644` (`list_script_to_diff/6`) together with `diff.ex:606` (`follow_snake/1`), reached via `diff_maybe_improper_list/3` (`diff.ex:399`) / `myers_difference_list/3` (`diff.ex:543`).
- **Issue** — When two lists are diffed, each *unequal* element pair is diffed **twice**: once in `follow_snake` (line 606) to test whether the Myers "snake" is equivalent, and again in `list_script_to_diff` (line 644) when the resulting `{:del}`/`{:ins}` pair is turned into an element-level diff. The result computed during the search is discarded rather than reused/memoised. For nested lists this doubling compounds multiplicatively, giving ≈2^depth cost. Equal structures avoid it (the snake result is reused as `{:eq, …}`), and tuples/maps avoid it (they diff each element/key exactly once), so it is specific to the list path.
- **Class** — Resource consumption (algorithmic complexity / super-linear cost from external input).
- **Impact** — A failing `assert a == b` (or `match?`) whose two sides are lists nested only ~23+ levels deep and differ anywhere causes `compute/3` to burn a full CPU core exponentially. The sole caller (`ExUnit.Formatter.find_diff/3`, formatter.ex:561) wraps it in `Task.yield(task, 1500) || Task.shutdown(task, :brutal_kill)`, so in normal use the effect is bounded to ~1.5 s of one core per such failing assertion, after which the diff is silently dropped (plain `inspect` shown instead). Where test fixtures/expected values derive from untrusted data (golden-file/JSON-array comparisons, property tests), an attacker who can make such an assertion fail forces the 1.5 s burn + degraded output per assertion. Called directly without the timeout wrapper, the hang is unbounded. No crash, no privilege crossing.
- **Evidence** — `verified`.
  - Growth is exponential (~1.95×/level): depth 12→22 measured `0.837, 3.078, 13.754, 53.821, 197.07, 748.962 ms` (×~3.8 per +2 depth).
  - Trigger requires per-level inequality: `compute(equal_depth_5000, equal_depth_5000, :==)` → `0.0 ms`; `compute(base_depth_24, other_depth_24, :==)` → `2972 ms`.
  - List-specific: at depth 40, nested tuples `0.014 ms`, nested maps `0.041 ms`, nested lists `TIMEOUT >3s`; a list-of-maps spine at depth 26 also `TIMEOUT`.
  - End-to-end via the formatter's exact `find_diff/3`: a 60-word input (list nested 30 deep) → `formatter find_diff wall time: 1502.2 ms`, `diff result: nil`.
- **Suggested fix** — Memoise element-pair diffs for the duration of a `compute` run (cache `diff/3` results keyed by the `(elem1, elem2)` pair), or thread the diff already produced in `follow_snake` through the edit script so `list_script_to_diff` reuses it instead of recomputing at line 644 — either collapses the cost from ≈2^depth to linear. A defence-in-depth depth/step bound in `myers_difference_list` (throwing `:undiffable`) would also cap it independently of the caller's timeout.

### Ruled out
- **`Code.eval_quoted` in `diff_guard_clause/2` (line 300)** — evaluates the developer-written guard AST (`x > 0`, `is_integer(x)`, …) placed in `when` position, with runtime values passed only as inert *bindings*. Verified a value bound to a guard variable is never executed as code (a `File.touch!` quoted into the value did not run; `is_integer(quoted_tuple)` → `false`); guard-BIF failures become `false`, not raises. Developer AST only, no value-code-exec.
- **`Code.eval_quoted` in `safe_eval/1` (line 1252)** — gated by `Macro.quoted_literal?(kw)` (line 735) before evaluation, so only a literal (no calls) from the left-side struct AST is evaluated; rescued regardless.
- **`Code.ensure_loaded?(struct)` / `struct.__info__(:struct)` / `struct!/2` (lines 781-794)** — the module atom comes from the left-side struct pattern AST (developer source, `%Foo{}`), never from value data; `load_struct` is rescued.
- **`inspect(value, safe: false)` in `safe_inspect/1` (line 1258)** — runs the compared struct's own `Inspect` implementation (project/dependency code that runs during any inspection), wrapped in `rescue`; no new sink and no value-controlled dispatch.
- **Atom creation from values** — none; verified 2000 distinct string values and a quoted-AST value produced an atom-table delta of `0`. No `String.to_atom`/`Module.concat`/`binary_to_term` in the file.
- **Value treated as code in `==`/`===` context** — verified a value that is the quoted AST of `IO.puts`/`System.cmd` is escaped and diffed structurally, not evaluated (no output, no file created).
- **`function_env_to_algebra/3` `"#Function<" <> contents = inspect(function)` (line 1153)** — reached only for two `:local` closures with identical module/name/uniq/index; `inspect` of such a fun always begins `#Function<`, so the match cannot fail.
- **`compute/3` crashing the formatter via the linked `Task`** — `diff_value/3` and `diff_quoted/5` both have catch-all clauses, so value inputs cannot raise `FunctionClauseError`; the only unrescued raise site is guard evaluation, which is developer AST and total in guard position.

## Risks / caveats
- **Flat-list and string diffs are quadratic** (`myers_difference_list`, `String.myers_difference`, `String.bag_distance`): two large, maximally-different flat lists/strings cost O(n²) (measured n=200/400/800 → 46/174/832 ms). This is the same property as public `String.myers_difference`, is bounded by the formatter's 1.5 s `brutal_kill`, and is subsumed by the exponential finding above; not reported separately.
- **The 1.5 s timeout is the primary mitigation.** The whole severity of the finding hinges on `find_diff/3` (formatter.ex:558-565) always wrapping `compute` in a killed `Task`. If a future refactor calls `ExUnit.Diff.compute/3` (which is `@doc`'d) directly, the blow-up becomes an unbounded hang — a follow-up should confirm no other call site exists (I found only formatter.ex:559).
- **`to_algebra/2` runs in the formatter process, not the time-boxed Task** (formatter.ex:506/512). I traced its clauses and found value inputs route through `inspect` (default `safe: true`) and AST through `Macro.to_string` (no eval); I did not find a crash there, but its cost is bounded only by the size of the diff structure `compute` produced within 1.5 s, not independently capped.
- I exercised `compute/3` directly (its documented signature). I did not drive the full `assert`-macro → `AssertionError` → formatter chain end-to-end through a real `mix test`; the AST shapes were reconstructed from the macro's documented `:original`/pattern conventions and match the clauses in `diff_quoted/5`.

