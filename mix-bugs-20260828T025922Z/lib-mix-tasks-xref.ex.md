# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/xref.ex
<!-- sha256: 6d68cb082fca491652f83b18a93b1c6dc9f904648efb219a2c5a3268b5f94bf1 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/xref.ex
<!-- sha256: 6d68cb082fca491652f83b18a93b1c6dc9f904648efb219a2c5a3268b5f94bf1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`mix xref` is largely correct. I confirm one verified correctness bug in the `--format stats --label compile-connected` outgoing-dependency counter, and one `@spec`/`@doc` contract mismatch on the deprecated public `calls/1`. The prior report's `calls/1` evidence was labelled `asserted` on the basis that the three sources (`@doc`, `@spec`, `add_call/5`) "were mutually inconsistent" without that being pinned down; all three live in this same file, so I re-read them directly and the evidence is now `verified`. No neighbour-module assumption was load-bearing for either finding — both are entirely local to `xref.ex`.

## Bugfixes

#### `CSCAN-xref-out-stats-compile-connected-overcount`
- **Finding ID:** CSCAN-xref-out-stats-compile-connected-overcount
- **Severity:** Medium
- **Confidence:** high
- **Location:** `elixir/lib/mix/lib/mix/tasks/xref.ex:1187`, `out_stats_filter/4`
- **Class:** State / accumulator threading (wrong node passed to filter predicate)
- **Impact:** `mix xref graph --format stats --label compile-connected` — the exact command the module's own `@moduledoc` recommends ("To verify the general health of your project… `mix xref graph --format stats --label compile-connected`") — reports an inflated "Top files with most **outgoing** dependencies" count. Any compile edge to a *leaf* file (a file with no further dependencies) is wrongly counted as compile-connected, even though the printed graph and the compile-connected definition exclude it. Users tuning recompilation hotspots are shown files as problematic sources when they are not, contradicting the very graph the same tool prints.
- **Evidence:** `verified`. Re-confirmed against the source: the `compile_connected` predicate is `fn {key, type} -> type == :compile and match?([_ | _], file_references[key] || []) end` (lines 1123-1126), and in `transitive_filter/2` (lines 1133-1139) it is applied to each child edge `{key, ...}` where `key` is the *target* of the edge. In `out_stats_filter/4` (lines 1180-1188) the code iterates `:digraph.out_neighbours(file)` (targets `v`) but calls `filter_fn.({file, label})` at line 1187 — passing the *source* `file` as `key` instead of the neighbour `v`. Because `references[file]` is always non-empty whenever `file` has any out-neighbour, the compile-connected check degenerates to "is a compile edge", ignoring whether the target is itself connected. `in_stats_filter/4` (lines 1193-1201) is *correct*: for an incoming edge `v → file` the target really is `file`, so `filter_fn.({file, label})` at line 1200 checks the right node.
  Reproduced with a 5-file project (`A --compile--> B(leaf)`, `C --compile--> D --runtime--> E`):
  ```
  $ mix xref graph --label compile-connected
  lib/c.ex
  └── lib/d.ex (compile)          # A→B correctly absent (B is a leaf)

  $ mix xref graph --format stats --label compile-connected
  Top 5 files with most outgoing dependencies:
    * lib/c.ex (1)                # correct
    * lib/a.ex (1)                # WRONG: should be 0 (A→B is not compile-connected)
    ...
  ```
  The graph omits `lib/a.ex` entirely, but stats reports it with 1 outgoing compile-connected dependency.
- **Suggested fix:** Pass the neighbour as the key: change line 1187 to `filter_fn.({v, label})`.

#### `CSCAN-xref-calls-spec-missing-caller-module`
- **Finding ID:** CSCAN-xref-calls-spec-missing-caller-module
- **Severity:** Low
- **Confidence:** high
- **Location:** `elixir/lib/mix/lib/mix/tasks/xref.ex:515-521`, `calls/1` `@spec`
- **Class:** Contract violation (`@spec` disagrees with implementation and `@doc`)
- **Impact:** Dialyzer consumes the `@spec` for the public (though `@deprecated`) `Mix.Tasks.Xref.calls/1`. The spec declares the return element as the closed map `%{callee: ..., line: ..., file: ...}`, but the implementation builds `%{callee:, caller_module:, file:, line:}`, and the `@doc` explicitly documents a fourth key `:caller_module`. The spec is thus narrower than the actual return, so any caller pattern-matching on the documented `:caller_module` key will be flagged as an impossible/typing error by Dialyzer.
- **Evidence:** `verified` (upgraded from `asserted` in first pass). All three sources are in this file and were re-read directly:
  - `@doc` (lines 502-507) lists four keys: `:callee`, `:line`, `:file`, `:caller_module`.
  - `@spec` (lines 515-521) lists only three: `callee`, `line`, `file` — no `caller_module`.
  - `add_call/5` (lines 613-622) populates exactly four keys: `callee`, `caller_module`, `file`, `line`.
  The `@spec` is the outlier: it omits the `:caller_module` key that both the documentation and the implementation carry.
- **Suggested fix:** Add `caller_module: module()` to the map type in the `@spec` so it matches the `@doc` and the implementation.

## Risks / caveats

- The stats over-count is a genuine (if narrow) upstream correctness bug in a documented, recommended command; the `calls/1` mismatch is likewise an upstream contract inconsistency.
- **Coverage limitation (not an assumption about a specific behavior):** I drove the `stats` path and the comprehension-filter semantics with live inputs, and read-reviewed the remaining graph-transformation helpers (`merge_groups`, `remap_references_to_group`, `sink_tree`/`source_tree`, cycle labeling) and found them consistent. I did not runtime-exercise every `--format`/`--source`/`--sink`/`--group` combination. This is a stated bound on coverage, not a hedged claim standing behind a dismissed finding — no specific defect in those unexercised paths is being asserted or ruled out on faith.

### Ruled out
- **`handle_callers` emitting `(nil)` type rows** — `verified`. The comprehension at lines 629-633 uses a `reference = reference(...)` assignment qualifier with no explicit truthiness guard. I confirmed via `elixir -e` that an assignment qualifier in an Elixir comprehension filters on the truthiness of its result (a `for x <- [1,2,3], y = (…nil…), do: …` yielded only the non-nil rows), so `reference/2` returning `nil` excludes that source rather than producing `[file, " (", nil, ")"]`. Executed evidence, not an invariant assumption. *Axes examined: nil-result-of-assignment-qualifier. Not examined: none.*
- **`in_stats_filter/4` compile-connected count** — `verified`. For an incoming edge `v → file` the target is `file`, so passing `{file, label}` at line 1200 checks the correct node; confirmed in the same 5-file repro that the leaf case reports incoming `lib/b.ex (0)` correctly. *Axes examined: source-vs-target-node for the compile_connected key on incoming edges. Not examined: none.*
- **`try/after` in `handle_trace/2` leaking the ETS table / compiler options on a compile exception** — no `rescue`/`catch` clause, so a raise propagates, but the `after` block (`:ets.delete(ets)` + `Code.compiler_options(old)`) always runs; cleanup is intact. This is a structural same-file claim about control flow, confirmed by reading; there is no input axis to try.
- **`:ets.delete(ets, :modules)` in `handle_trace/2`** — deletes a key that is never inserted; a harmless no-op, not a behavioral defect. Structural same-file claim; no input axis to try.

### Did I verify this?

- **`out_stats_filter` overcount (line 1187)** — Yes. Re-read lines 1119-1202 and confirmed source-vs-target key confusion; backed by a live 5-file repro showing `lib/a.ex (1)` in stats while absent from the graph.
- **`calls/1` `@spec` mismatch** — Yes, and upgraded from `asserted` to `verified`: `@doc` (502-507), `@spec` (515-521), and `add_call/5` (613-622) all read directly in-file; the `@spec` is the sole source missing `:caller_module`.
- **`handle_callers` nil rows** — Yes, executed with `elixir -e` (comprehension qualifier filters falsy results).
- **`in_stats_filter` correctness** — Yes, confirmed by the same live repro (incoming leaf count `0`).
- **`try/after` cleanup and `ets.delete(ets, :modules)` no-op** — Yes, by reading the local control flow; these are structural claims with no input axis to exercise.

### Dropped in re-review

None. Every finding and ruled-out entry from the first pass is preserved; the only change is upgrading the `calls/1` evidence label from `asserted` to `verified` after re-reading the three in-file sources.
