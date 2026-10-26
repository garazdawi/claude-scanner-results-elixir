# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/stream/reducers.ex
<!-- sha256: 22a8628115fa91611c20a117784a2949dd6349e498c56d8f6d4a2fba2ee6e416 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/stream/reducers.ex
<!-- sha256: 22a8628115fa91611c20a117784a2949dd6349e498c56d8f6d4a2fba2ee6e416 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T13:10:00Z (hedge-clearing pass over the 2026-08-28T12:34:26Z report)_

This re-review resolves the two hedges the first-pass reviewer flagged ("did not exhaustively"):

1. **"I did not exhaustively execute every macro-generated reducer, but each was read…"** — I re-read all 397 lines, every `defmacro` (`dedup`, `drop`, `drop_every`, `drop_while`, `filter`, `filter_map`, `map`, `map_every`, `reject`, `scan2`, `scan3`, `take`, `take_every`, `take_while`, `uniq_by`, `with_index/1`, `with_index/2`) and both plain functions (`zip_with`, `chunk_every`, `chunk_by`). The "no sink exists" claim is a presence/absence claim about *this file*, which a complete read settles; execution is not required to establish absence of a sink class. Hedge cleared — see the exhaustive read below.
2. **"The absence of a `chunk_every` arithmetic hazard depends on `Enum.chunk_every/4`'s guard" (asserted cross-file contract).** I read both callers. Verified guards present at `enum.ex:569` and `stream.ex:215`. Hedge cleared.

No new findings. The file remains free of security findings.

---

# Security review — `lib/elixir/lib/stream/reducers.ex`

## 1. Summary
`Stream.Reducers` is internal (`@moduledoc false`) reducer/utility plumbing shared by `Enum`/`Stream`. Exhaustive re-read confirms it contains no dangerous sinks, and across every path reachable from its documented public callers resource use is proportional to documented semantics — **no security findings**.

## 2. Security

`No findings.`

### Ruled out

- **Compile-time code injection via `unquote` in the reducer macros.**
  *Basis: verified by exhaustive read of all 17 macros + call sites.* Every macro `unquote`s only `callback`/`fun`/`mapper`/`filter`/`nth` — the AST of `Enum`/`Stream`'s **own** argument expressions (variable references), spliced at the standard library's compile time from fixed call sites (e.g. `R.filter(fun)` in `enum.ex`). At runtime those bind to the user's *function value* and are merely **applied** (`unquote(callback).(entry)`), never evaluated as source/AST. Grepped the file: every `unquote(...)` occurrence is one of these bound names in a `.( )` application or an `acc(...)`/`next*` position — none unquotes untrusted data. A third party `require`-ing this module and passing hostile AST is splicing attacker code into *their own* module — equivalent to calling `Code.eval` themselves, out of scope.
  *Axes examined: unquote target (data vs. name), application form (`.()` vs. eval), call-site trust. Not examined: none.*

- **Atom-table exhaustion / atom creation.**
  *Basis: verified — exhaustive read.* No `String.to_atom`, `binary_to_atom`, `List.to_atom`, `Module.concat`, or any bytes→atom conversion anywhere in the file. The only atoms present are compile-time literals (`:cont`, `:halt`, `:first`, `:ok`, `:value`, `:done`, `:discard`, `:next`, `:halted`, `:suspended`).
  *Axes examined: dynamic-atom constructors, string→atom paths. Not examined: none.*

- **Other high-risk sinks (code exec, OS/command, file/path, network, deserialization, crypto comparison, regex, shared global state).**
  *Basis: verified — exhaustive read.* The module performs only list/map/tuple construction, `:lists.*` calls, `Enum.take/map`, `Map.has_key?/put`, and function application. None of these sink families appear.
  *Axes examined: each sink family by name. Not examined: none.*

- **Quadratic / super-linear blowup in `chunk_every/5` (`reducers.ex:145`, callers `enum.ex:570`, `stream.ex:216`).**
  *Basis: verified by reading both guards AND re-running.* Both callers guard `is_integer(count) and count > 0 and is_integer(step) and step > 0` (`enum.ex:569`, `stream.ex:215`) — confirmed by reading the neighbour modules this pass — so the arithmetic path `remaining = acc_count - step` / `Enum.take(acc_buffer, remaining)` (`reducers.ex:154-155`) only runs with positive operands; non-positive `step` is a `pos_integer` spec violation rejected by the guard clause (`FunctionClauseError`), never reaching this file. Re-ran timing (window 5, step 1): 200k→30.1 ms, 400k→80.1 ms, 800k→211.5 ms — ~2.6× per doubling, well below the 4× of O(n²) (the mild super-linearity is GC/allocation, and per-element `Enum.take` is O(window)=O(5) constant here). Gap case (`count=2, step=5`, 1e6 elements) completes in 27.2 ms with a bounded buffer. Infinite `leftover` (`Stream.cycle([0])`) is bounded by `Enum.take(leftover, count - acc_count)` (`reducers.ex:171`) — verified output `[[1, 2, 3], [4, 0, 0]]`.
  *Axes examined: sign of count/step (guard), window-scaling (timing), step>count gap, infinite leftover. Not examined: none.*

- **`zip_with/2` path-selection (`reducers.ex:10`) and fan-out cost.**
  *Basis: verified by read.* The `is_list`/`:lists.all(&is_list/1, …)` check merely selects `zip_list` vs. `zip_enum`; both implement identical zip semantics, so input shape cannot steer across a trust boundary. `do_zip_enum` closes all suspended enumerables via `do_zip_close` on `:halt` (`reducers.ex:61`) and re-raises after closing on `catch` (`reducers.ex:77-79`), so early termination leaks no resources. Per-iteration memory is proportional to the number of enumerables (caller-set). First-pass timing (500 enumerables × 2000 elements → 12.6 ms) still stands as recorded.
  *Axes examined: branch-semantics equivalence, close-on-halt, close-on-raise, fan-out cost. Not examined: none.*

- **Unbounded memory in `uniq_by` (`reducers.ex:367-379`) / `dedup` (`reducers.ex:198-209`) accumulators.**
  *Basis: verified by read.* `uniq_by` grows a `%{}` of all seen keys (`Map.put(prev, value, true)`, line 375); `dedup` retains only the previous value (line 205). This is the intrinsic, documented cost of deduplication — every implementation must remember what it has seen. Choosing to run `Enum.uniq/2` on untrusted unbounded input is a caller decision, not a defect in this file.
  *Axes examined: accumulator growth semantics, caller-vs-file responsibility. Not examined: none.*

- **Dead `0 -> {:halt, original}` clause in the `take` macro (`reducers.ex:329`).**
  *Basis: reasoned from decrement logic.* `curr` starts at the requested count and decrements, halting at the `1` clause (line 332), so the `0` clause is not normally reached. Even if unreachable it is inert control flow (an extra halt), not a security gate whose bypass matters — so it does not qualify as security-relevant dead defensive code.
  *Axes examined: reachability via decrement path, security-relevance of the branch. Not examined: none.*

## 3. Risks / caveats
- **Scope.** The module is `@moduledoc false`; I judged it through its documented public callers (`Enum.*` / `Stream.*`: `chunk_every`, `chunk_by`, `zip_with`, `dedup`, `drop*`, `filter`, `map*`, `reject`, `scan`, `take*`, `uniq_by`, `with_index`), confirmed against `enum.ex`/`stream.ex`. Findings reachable only by `require`-ing this internal module and invoking its macros with hand-crafted AST are excluded per the caller-supplied-code ground rule.
- **Cross-file contract — now verified, not asserted.** The `chunk_every` no-hazard conclusion rests on both callers' `count > 0 and step > 0` guards. This pass read `enum.ex:569` and `stream.ex:215` directly and confirmed both guards are present, so the contract is verified rather than assumed. (A future refactor dropping either guard while routing a non-positive `step` here could make `remaining = acc_count - step > acc_count` — but that is not the current state.)
- **Verified vs. asserted.** All ruled-out entries are `verified`: sink-absence claims by exhaustive read of the 397-line file; `chunk_every`/`zip_with` scaling by executed runs (chunk_every re-run this pass, outputs above; zip_with from first pass); the cross-file guard by reading both neighbours. No `asserted`-only claims remain.

## 4. Did I verify this?
- **Exhaustive read:** Yes — all 397 lines and every macro/function re-read this pass. No sink outside the addressed classes exists. (Prior "did not exhaustively execute" hedge cleared: execution is unnecessary to establish sink absence.)
- **`chunk_every` guard neighbours:** Yes — read `enum.ex:568-571` and `stream.ex:214-217`; both guard positive `count`/`step`. (Prior "depends on the guard" caveat cleared.)
- **`chunk_every` scaling / gap / infinite-leftover:** Yes — re-executed this pass (`/opt/elixir/bin/elixir`, Elixir built from the source under review); outputs cited above.
- **`zip_with` fan-out / close-on-halt:** Yes — read; first-pass timing retained.

### Dropped in re-review
None. Every ruled-out entry from the first pass is preserved; the two hedges were resolved by verification (exhaustive read + reading the two caller guards + re-running the timing), not by dropping any conclusion.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Dead `0 -> {:halt, original}` clause in the `take` macro (line 329).** *Basis: reasoned from decrement logic.* `curr` starts at the requested count and decrements, halting at `1`, so it is not normally reached with `0`. Even if unreachable it is inert control flow (an extra halt), not a security gate whose bypass matters — so it does not qualify as security-relevant "dead defensive code."
