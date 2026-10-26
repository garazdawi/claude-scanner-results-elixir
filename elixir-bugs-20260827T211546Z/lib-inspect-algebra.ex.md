# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/inspect/algebra.ex
<!-- sha256: 2c43d2ecba2b70adc62fc01b07406fc6f070090aa37107dc5fc10a209933ab7d -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/inspect/algebra.ex
<!-- sha256: 2c43d2ecba2b70adc62fc01b07406fc6f070090aa37107dc5fc10a209933ab7d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (hedge-resolution pass over the 2026-08-27 first pass)._

The review target path did not exist in-tree this pass; the installed copy at `/opt/elixir/lib/elixir/lib/inspect/algebra.ex` is **byte-identical** (sha256 `2c43d2ec…933ab7d`, matching the recorded header), so every claim below was re-checked against that copy and executed under Elixir 1.21.0-dev / OTP 29.

**Resolution of the two flagged hedges.** Both flagged phrases attach to the single `fits?`/`format` ruled-out entry. Per protocol I first asked whether the standing-context neighbour modules could bear on them: they are the compiler front-end `.erl` sources (`elixir_tokenizer`, `elixir_expand`, `elixir_config`, …). `Inspect.Algebra` is a pure pretty-printer with no dependency on any of them — they are not call-graph neighbours, and (per the protocol) reading a neighbour cannot clear a `Not examined` axis anyway. So I resolved the crash-safety axis the only admissible way: by **execution** (re-ran the 20k-iteration randomized document fuzz, 0 escapes). The two remaining axes (width-optimality and the `doc_limit(x,:infinity)` reversion boundary) require a second implementation to differentially disagree with; none exists in-tree, so I could not try them, and they stay honestly on `Not examined` with the entry `asserted`. No axis was upgraded on the strength of a re-read.

## Summary

`container_each/5` drops the updated `Inspect.Opts` when inspecting the tail of an improper list, so the shared `:limit` budget is under-counted by one and `inspect/2` prints one element more than `:limit` permits whenever an improper list is present. Re-confirmed by execution this pass.

## Bugfixes

#### `CSCAN-algebra-improper-tail-opts-dropped`
- **Finding ID** — `CSCAN-algebra-improper-tail-opts-dropped`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/inspect/algebra.ex:594-600`, `container_each/5` (improper-list clause); specifically line 596 (`{right, _opts} = ...`) and the `opts` returned on line 599.
- **Issue** — When the collection is an improper list `[left | right]`, the clause calls `call_container_fun` twice. The first call rebinds `opts` (limit decremented for `left`), but the second call's returned options are discarded (`{right, _opts} = ...`), and the clause returns the *pre-tail* `opts`. Every other path — the proper-list clause (line 590-591) and `call_container_fun` itself (lines 610-617) — threads the decremented `opts` forward. The tail element is rendered (` | right`) but never charged against the limit budget, and any `custom_options`/`limit` changes it produced are lost.
- **Class** — State / accumulator threading (dropped accumulator across a return step).
- **Impact** — The documented `:limit` of `Kernel.inspect/2` (and the "updated options" contract of `Inspect.Algebra.container_doc_with_opts/6`) is not honored precisely: any structure containing an improper list shows one extra item before `"..."`. Cosmetic/diagnostic only — no crash, no data loss — but it makes `:limit`-bounded output size differ between the same data written as an improper vs. proper list.
- **Evidence** — `verified` (re-executed this pass). Ran under Elixir 1.21.0-dev / OTP 29:
    ```
    [[1 | 2], :a, :b, :c, ...]
    [[1, 2], :a, :b, ...]
    {[1 | 2], :a, :b, :c, ...}
    {[1, 2], :a, :b, ...}
    ```
    The improper-list variant prints `:c`, one element past where the equivalent proper list truncates, for both list and tuple containers. The source lines 594-600 are present exactly as read.
- **Suggested fix** — Thread the tail's options forward instead of discarding them:
  ```elixir
  defp container_each([left | right], opts, fun, acc, simple?) do
    {left, opts} = call_container_fun(fun, left, opts)
    {right, opts} = call_container_fun(fun, right, opts)   # was {right, _opts}
    simple? = simple? and simple?(left) and simple?(right)
    doc = join(left, right, simple?, @tail_separator)
    {:lists.reverse([doc | acc]), simple?, opts}
  end
  ```

## Risks / caveats
- This behavior is present in the released tree (byte-identical to the installed `Inspect.Algebra`), so it may be viewed as long-standing/accepted rather than a regression. I classify it as a real defect because the proper-list path threads the accumulator and the improper path is the only asymmetric exception.

### Ruled out
- **All 25 doctests in `Inspect.Algebra`/`Inspect.Opts`** — re-ran under ExUnit `doctest` this pass; `Result: 25 passed`. Basis: `verified`. Axes examined: documented input/output pairs (executed) | Not examined: none.
- **`no_limit/1` doc_limit round-trip** (`format`/`fits?` lines 1271-1275, 1368-1374) — exercised via the doctest set (documented example + width restore). Basis: `verified`. Axes examined: documented example + finite→:infinity→finite width restore (executed) | Not examined: none.
- **`container_doc_with_opts` `:limit`/`:separator`/`:break` doctests** — reproduced `"[1,\n 2,\n 3,\n 4,\n 5]"`, `"[1, 2, 3, ...]"`, `"[1! 2! 3! ...]"` via the doctest run. Basis: `verified`. Axes examined: the three documented option shapes (executed) | Not examined: none.
- **`call_container_fun` `limit <= 0`/`:infinity` guard and `doc_nil` non-decrement** (lines 602-618) — `:infinity <= 0` is `false` (atom outranks integer in Erlang term order), so the guard short-circuits only on genuine non-positive integer limits; empty docs deliberately don't consume budget (line 615). Confirmed against the source lines and the proper-list threading exercised by the fuzz. Basis: `verified`. Axes examined: `:infinity` vs non-positive-integer boundary, `doc_nil` no-decrement, proper-list threading (executed) | Not examined: none.
- **`fits?`/`format` fixed-point machinery for pathological documents** (lines 1252-1374, incl. `doc_limit`/`:optimistic`/`doc_force`/flex-break interactions) — re-ran a 20,000-iteration randomized fuzz (seeded `:exsss`, `{11,22,33}`) generating nested documents from `string`/`concat`/`glue`/`group`/`nest`/`group(:optimistic)`/`no_limit`/`line`, formatted at widths `{5, 10, 20, :infinity}` through the public `Inspect.Algebra.format/2` and forced to `iodata_to_binary`; **0 escaping exceptions**. This is the crash-on-pathological-nested-input axis, now executed. Basis: `asserted`. Axes examined: crash-safety under randomized nesting of `:optimistic` groups + `no_limit`/`doc_limit` + flex breaks + hard lines across finite and `:infinity` widths (executed, 20k iters) | **Not examined**: width-optimality / minimal-break correctness (no independent pretty-printer oracle exists in-tree to differentially compare against — the axis cannot be *tried*, so it cannot be cleared), and the exact `doc_limit(x, :infinity)` reversion boundary under interleaved nesting depth (same reason — no second implementation to disagree with). These two remain on `Not examined` and the entry stays `asserted` because neither could be executed; the neighbour `.erl` modules are not call-graph neighbours and reading them would not clear them.
- **`group/2` `:self`/`:inherit`/invalid-mode `case`** (lines 1054-1064) — invalid modes raise `CaseClauseError`, which is misuse outside the `@spec` domain, not a contract violation. Basis: `asserted`. Axes examined: the three documented modes (via passing doctests) | Not examined: arbitrary non-atom mode arguments (spec-domain input, out of contract — not exercised).
- **`color_doc/3` falsy-color skip** (line 664, `if precolor = Keyword.get(...)`) — colors are `IO.ANSI.ansidata` (never legitimately `nil`/`false`), so the truthiness guard cannot drop a valid color; the `:reset` default resolves via `ansi/1`. Basis: `asserted`. Axes examined: documented `syntax_colors` shapes | Not examined: a caller injecting `false`/`nil` as a color value (silently skipped, but outside the documented color-value domain — not exercised).

### Did I verify this?
- **Main finding (`CSCAN-algebra-improper-tail-opts-dropped`)** — Yes, `verified` by re-execution: `inspect(..., limit: 5)` prints one extra element for improper lists vs the equivalent proper list, across both list and tuple containers. Source lines 594-600 present as read.
- **`fits?`/`format` machinery** — Partially. Crash-safety under randomized nested/optimistic/limit/hard-line documents is `verified` by executed fuzz (0 escapes / 20k iters, re-run this pass). Width-optimality and the `doc_limit(:infinity)` reversion boundary are **not examined**: there is no second implementation in-tree to differentially disagree with, so these axes could not be *tried*; per protocol a re-read cannot substitute for trying them, so they stay untried and the entry stays `asserted`. No divergence surfaced on any axis I could execute.
- **Doctests and the other ruled-out entries** — Yes, `verified` by execution as noted per entry (doctests re-run this pass, 25 passed).

### Dropped in re-review
- None. Every first-pass finding and dismissal is preserved.
  - The two flagged first-pass hedges ("did not exhaustively" / "not fuzzed") were not findings — they were caveats on the `fits?`/`format` dismissal. This pass replaces them with an explicit axis ledger: the crash axis is executed (20k-iter fuzz, 0 escapes), and the two unexecutable axes (width-optimality, `doc_limit(:infinity)` boundary) are named on `Not examined` with the reason they cannot be tried (no in-tree oracle). The entry remains a dismissal at `asserted` — nothing was upgraded to a finding, because no executed axis produced a divergence, and nothing was upgraded to `verified`, because two axes could not be executed.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

  - **Upgraded/clarified from hedged assumption in first pass**: the first pass said "my search there was structural, not fuzzed." The crash axis is now executed; the remaining un-tried axes are named explicitly above rather than left as a blanket hedge. No divergence was found on the axes I could run, so this stays a dismissal (not upgraded to a finding); the two named axes remain honestly on `Not examined` because no oracle exists to try them against.
