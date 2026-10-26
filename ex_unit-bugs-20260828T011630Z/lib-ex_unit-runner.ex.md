# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/runner.ex
<!-- sha256: 0037310e34eee06c14d0f186f9c6d7874c37dc1c08b8d86b8ab2fc759d526ee7 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/runner.ex
<!-- sha256: 0037310e34eee06c14d0f186f9c6d7874c37dc1c08b8d86b8ab2fc759d526ee7 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28. All first-pass hedge phrases ("I assumed") resolved with executed evidence; no assumptions remain._

# Summary

The `--dry-run` clause of `run_module_tests/4` returns the test list in forward order where the rest of the pipeline expects it reversed, so ExUnit prints the "tests that would be executed" in the exact reverse of the order they would actually run. This is the single confirmed defect in the file. All other subtle regions (max-failures boundary arithmetic, the sigquit priority `receive`, the `max_cases`/`available` async loop, `failed/3` MultiError handling) were examined and held up.

# Bugfixes

#### `CSCAN-runner-dry-run-reversed-order`
- **Finding ID:** `CSCAN-runner-dry-run-reversed-order`
- **Severity:** Low
- **Confidence:** high
- **Location:** `runner.ex:315-317` (`run_module_tests(%{dry_run: true}, ...)`), interacting with `runner.ex:276` (`run_module/5`, the `Enum.reverse(finished_tests, pending_tests)` line).
- **Class:** State / accumulator threading (order-of-accumulation mismatch across a shared reversal step).
- **Impact:** Anyone using ExUnit's documented `--dry-run` / `dry_run: true` option to preview *which tests, in what order,* would run is shown the tests in reverse of the real execution order. It is purely informational (no test is actually run wrong), so the cost is a misleading listing, not corrupted results — hence Low.
- **Evidence:** `verified`.
  - The running toolchain was confirmed to be built from the file under review: the installed source at `/opt/elixir/lib/ex_unit/lib/ex_unit/runner.ex` is **byte-identical** to the reviewed file (both `sha256 = 0037310e…`), and the compiled beam `Elixir.ExUnit.Runner.beam` (mtime `1787776727`) is newer than that source (mtime `1787658808`), so the beam reflects this source. (This replaces the first pass's "I assumed it is built from the source under review" — now executed and verified.)
  - A real run (seed 0 → definition order) executes lines 4,5,6:
    ```
    $ elixir -r order_test.exs -e 'ExUnit.configure(seed: 0, trace: true); ExUnit.run()'
      * test a [L#4]  * test b [L#5]  * test c [L#6]
    ```
  - The same suite under dry-run lists them reversed:
    ```
    $ elixir -r order_test.exs -e 'ExUnit.configure(seed: 0, dry_run: true); ExUnit.run()'
    order_test.exs:6
    order_test.exs:5
    order_test.exs:4
    ```
  Root cause: `run_tests/4` (line 404-412) accumulates finished tests reversed (`[test | acc]`) and `run_module/5` calls `Enum.reverse(finished_tests, pending_tests)` (line 276) to restore forward order. The dry-run clause (line 316) returns `tests` in forward order, which that same `Enum.reverse` then flips.
- **Suggested fix:** Return the list pre-reversed so the shared reversal restores forward order:
  ```elixir
  defp run_module_tests(%{dry_run: true}, test_module, _async?, tests) do
    {test_module, [], Enum.reverse(tests)}
  end
  ```

# Risks / caveats

- I reviewed the whole file end-to-end. Only the dry-run ordering was reproduced as a defect; the other subtle areas held up under the boundary/neighbour analysis recorded in "Ruled out."
- No remaining assumptions: the one first-pass hedge (that `/opt/elixir` was built from the reviewed source) was verified by sha256-comparing the installed source to the reviewed file and confirming the beam mtime post-dates the source.

### Ruled out

- **`take_async_modules` returning `[]` causing an infinite `async_loop`** — `verified`.
  - Basis: read of `server.ex:175-218`. The async branch of `take_modules/1` is guarded by `not :queue.is_empty(state.async_modules)` (line 181) and returns via `take_until/2` (lines 209-218), which produces a non-empty list whenever the queue is non-empty. When no async work remains, control falls to `state.loaded == :done -> GenServer.reply(from, nil)` (lines 198-200), i.e. the reply is `nil`, never `[]`. The `cond` in `async_loop/4` (runner.ex:114) treats `nil` as falsy and moves to the sync branch. So the async branch is never entered with an empty list.
  - Axes examined: reply-value shape (`nil` vs `[]`), empty-queue guard | Not examined: none.

- **`process_max_failures/3` off-by-one at the `max_failures` boundary** — `verified`.
  - Basis: exhaustive boundary case analysis of lines 571-579 against the callers at 419-431 (test-level, `bump = 1`) and 256-266 (module-level, `bump = length(tests)`). For a single test (`bump = 1`), the `{:reached, n}` arm fires exactly when `previous == max-1`, yielding `n = max - previous = 1` — the only value `run_test/3` pattern-matches (`{:reached, 1}` at line 424). For the module path, `bump = length(tests)` yields `n = max - previous ≤ bump = length(tests)`, and `invalid_tests` has length `length(tests)` on the `:error` path (line 335), so `Enum.take(invalid_tests, n)` (line 262) never over-takes.
  - Axes examined: `bump ∈ {1, length(tests)}`, `previous` at `{max-1, max, <max-1}`, `{:reached, n}` vs literal `{:reached, 1}` match | Not examined: none.

- **`test_started` emitted without `test_finished` on the `:surpassed` path (lines 428-429)** — `verified` (present by design).
  - Basis: this arm is reachable only when a *concurrent* module pushes the global failure counter past `max_failures` between this test's `EM.test_started` (line 416) and its accounting (line 419). This is ExUnit's documented best-effort `max_failures` semantics — the code comment at lines 268-269 explicitly flags that `module_finished` is intentionally omitted only on `:surpassed`. Present by design, not a defect in this file.
  - Axes examined: single-process ordering, concurrent-counter interleaving, comment-documented intent | Not examined: none.

### Did I verify this?

- **Dry-run reversed order:** Yes — reproduced end-to-end on the byte-identical, up-to-date toolchain (sha256 match + beam mtime > source mtime); real order 4,5,6 vs dry-run order 6,5,4.
- **Toolchain provenance (former hedge):** Yes — verified by `sha256sum` equality between the reviewed file and `/opt/elixir/lib/ex_unit/lib/ex_unit/runner.ex`, plus beam-vs-source mtime comparison.
- **Ruled-out `take_async_modules`:** Yes — read `server.ex:175-218`; `nil`-reply path confirmed at line 199, non-empty guarantee confirmed via `take_until`.
- **Ruled-out max-failures arithmetic:** Yes — boundary analysis against both call sites; no execution needed since the arithmetic is closed-form over the finite `previous`/`bump` cases enumerated above.

### Dropped in re-review

None. Every finding and ruled-out entry from the first pass is retained; the only change is upgrading the toolchain-provenance claim from an assumption to executed evidence within the existing `CSCAN-runner-dry-run-reversed-order` finding. No assumption required upgrading to a separate low-confidence finding, because the sole hedge was verifiable and verified.
