# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/history.ex
<!-- sha256: 737ba873c95bb6651b86568551f3f2aa9dba2fff3497e635b448f57b8a08ec5c -->

_Analysed on 2026-08-27T07:54:06Z_

## Summary
`IEx.History` is a self-contained evaluation-history data structure whose only inputs are interactive-session evaluation results, the `v(n)` index, and operator config — none of which cross a trust boundary into a dangerous sink, so no security defect is present.

## Security

No findings.

All externally-influenced values entering this file were traced to session-local / operator-controlled origins, and the file contains none of the dangerous sink shapes (no `eval`/compile, no atom creation, no path/file/command/network operations, no deserialization, no external-type dispatch):

- `entry` in `append/3` is `{line, result}` (`evaluator.ex:357`), where `result` is the value of code the user already evaluated at the prompt — self-inflicted, and getting a value into history *requires* prior code execution.
- `n` in `nth/2` is the argument to `v(n)` (`helpers.ex:460`), typed by whoever drives the shell — the same principal that has full RCE.
- `limit`/`start` come from `IEx.Config.history_size()` and a line counter — operator config.

The only side-effecting operation, `collect_garbage/0`, garbage-collects `:user` and the group leader and is wrapped in a `try/catch`; it takes no external input.

### Ruled out
- **Float / non-integer `v(n)` crashes `get_nth`** — `nth(state, 1.5)` passes the first `nth` guard (`1.5 - start` is a valid number) and reaches `get_nth(q, 0.5)`, which matches neither `0` nor `n > 0` after decrement and raises `FunctionClauseError`. Verified (`v(1.5) raises: FunctionClauseError`). Not a finding: `n` originates from the interactive caller (`v/1` is documented to take an integer), and crashing the evaluator is an expected, handled path — no network/disk/dependency data reaches it.
- **`has_binary/1` propagating a non-`:found` exception** — the `try/catch` only catches `throw :found`; any other exception from `has_bin` would escape. Ruled out: the traversed term is a self-constructed evaluation result, and I confirmed nested tuples, improper-looking data, bitstrings, and maps traverse without an uncaught crash (`append/nth OK, no uncaught crash on odd terms`).
- **`has_bin/1` traversal as a DoS** — it walks the full entry on eviction. Cost is O(term size) of a term the user themselves constructed; list/tuple breadth is handled by tail calls (only nesting depth uses stack), and BEAM terms cannot be cyclic, so no unbounded loop. Self-inflicted, not cross-boundary.
- **`has_bin/1` not descending into maps** — a large binary held inside a map won't trigger `collect_garbage`. This is a GC-heuristic/correctness gap (matching the referenced upstream `shell.erl`), not a security issue.
- **`raise` string interpolation of `n`/`start` in the `nth` out-of-bounds clauses** — the interpolated values land in a `RuntimeError` message shown back to the same interactive user; not a log/format/query interpreter sink.

## Risks / caveats
- Review was limited to this file. `IEx.History` is `@moduledoc false` (internal); its safety depends on callers (`evaluator.ex`, `helpers.ex`) continuing to feed it only session-local evaluation results and integer indices — those call sites, and the broader "who can reach the group leader / distribution port" boundary, are out of scope here and covered by the folder threat model.
- Behavioral claims above were exercised against `elixir`/`iex` built from the source under review; the float-index crash and odd-term traversal were run and observed.

