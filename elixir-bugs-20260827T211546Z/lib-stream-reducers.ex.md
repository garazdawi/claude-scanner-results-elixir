# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/stream/reducers.ex
<!-- sha256: 22a8628115fa91611c20a117784a2949dd6349e498c56d8f6d4a2fba2ee6e416 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/stream/reducers.ex
<!-- sha256: 22a8628115fa91611c20a117784a2949dd6349e498c56d8f6d4a2fba2ee6e416 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (first pass 2026-08-27T23:44:54Z). Every "executed" basis below was **re-run from scratch** in this pass, not carried over._

## Summary

`Stream.Reducers` (`@moduledoc false`) is the reducer engine behind the public `Enum.*`/`Stream.*` functions. This is compiler-adjacent library code, not a network-protocol surface; the folder threat model's tokenizer/interpolation/Unicode obligations (claims 1–16, RFC clauses 1–42) do **not** apply to this file — it contains no identifier tokenization, no UCD table parsing, no bidi/NFC/confusable logic, and no config/gen_server state. It is pure reducer plumbing over the caller's accumulator.

The sole first-pass hedge concerned the suspend/resume/halt paths of `zip_list`/`do_zip_enum`. In this pass I re-executed those paths by hand-stepping the `Enumerable.reduce` protocol (`{:suspend,…}`, `{:cont,…}`, `{:halt,…}` acc tuples) against both the all-list and enum implementations, plus a `Stream.resource` cleanup probe. All observed returns matched the eager reference. **No genuine, reachable bug found.**

## Bugfixes

No findings.

## Risks / caveats

- The module is `@moduledoc false` and reachable only indirectly; every behavior must manifest through a public `Enum`/`Stream` function. Behavioral verification (re-executed this pass) covers `chunk_every` (overlap/skip/`:discard`), `zip_with` (list + enum paths, uneven lengths), `dedup`, `scan/2`, `scan/3`, `uniq_by`, `take_every`, `map_every`, `drop_every`, and the zip suspend/resume/halt protocol.
- This module is stateless (pure functions over the caller's acc; no shared mutable state, no ETS, no process state), so "concurrent composition" is not a distinct behavioral axis — correctness depends only on the reduce-protocol contract, which was exercised for `:cont`/`:suspend`/`:halt` in both implementations.

### Ruled out

- **Stale comment in `do_zip_next/6` (lines 113–117) says it returns `{:halted, acc}` but the code returns `{:done, acc}`.** Not a bug: `do_zip_enum`'s `else` clause matches `{:done, _acc} = other` and returns it unchanged (lines 84–85); `{:done, acc}` on stream termination is correct. Comment is stale but internal (not `@doc`/doctest), no behavioral effect.
  - Basis: **verified (executed, re-run this pass)**. Enum-path suspend stepping of `Stream.zip_with([Stream.map(1..3,…), Stream.map(10..30//10,…)], sum)` returned `{:suspended, [11], <do_zip_enum cont>}` then `{:halted, [11]}` on `{:halt,…}`; full `Enum.to_list` = `[11,22,33]`; uneven `[1..5, 10..20//10]` terminated at `[11,22]` == eager `Enum.zip_with` = `[11,22]`.
  - Axes examined: `:cont`/`:suspend`/`:halt` re-entry, uneven-length termination, resource cleanup on halt, list-path vs enum-path. | Not examined: none.
- **`chunk_every` overlap (`step < count`) retains wrong elements** via `Enum.take(acc_buffer, acc_count - step)` on the reversed buffer. Ruled out: `Enum.chunk_every(1..10, 3, 2)` → `[[1,2,3],[3,4,5],[5,6,7],[7,8,9],[9,10]]`, correct overlap.
  - Basis: **verified (executed, re-run this pass)**. Axes examined: overlap step, buffer-reversal order. | Not examined: none.
- **`chunk_every` skip (`step > count`) mis-drops elements.** Ruled out: `Enum.chunk_every(1..10, 2, 3, :discard)` → `[[1,2],[4,5],[7,8]]`, correct.
  - Basis: **verified (executed, re-run this pass)**. Axes examined: skip step, `:discard` leftover. | Not examined: none.
- **`drop_every`/`map_every`/`take_every` off-by-one on the `:first` seed vs `nth`.** Ruled out: `drop_every(1..10, 3)` → `[2,3,5,6,8,9]`; `map_every(1..10, 3, *100)` → maps positions 1,4,7,10 (`[100,2,3,400,5,6,700,8,9,1000]`); `take_every(1..10, 3)` → `[1,4,7,10]`. All correct. `nth == 0` is short-circuited in `Enum` before reaching these macros.
  - Basis: **verified (executed, re-run this pass)**. Axes examined: `:first` seed, `nth` boundary. | Not examined: none.
- **`scan/2` (`scan2`) first-element handling and `dedup`/`uniq_by` accumulator threading.** Ruled out: `scan(1..5, +)` → `[1,3,6,10,15]` == `scan(1..5, 100, +)` shifted appropriately; `dedup([1,1,2,2,3,1,1])` → `[1,2,3,1]`; `uniq_by([1,2,3,2,1], &rem(&1,10))` → `[1,2,3]`.
  - Basis: **verified (executed, re-run this pass)**. Axes examined: `:first` seeding, acc threading. | Not examined: none.

### Did I verify this

Yes — re-executed this pass (not carried from the first pass):

1. **`zip_list` (all-list path).** Hand-stepped the reducer with a suspending reducing-fun: initial `{:suspend,[]}` → `:suspended`; one resume → `{:suspended, [11], <zip_list cont>}`; then `{:halt,…}` → `{:halted, [11]}`. Eager reversed reference `[33,22,11]` matches the cont-only order.
2. **`do_zip_enum` (non-list path, ranges + `Stream.map`).** Same stepping → `{:suspended, [11], <do_zip_enum cont>}`, then `{:halted, [11]}` on halt; full run `[11,22,33]`; uneven lengths terminate at `[11,22]`, equal to the eager `Enum.zip_with` reference.
3. **Resource cleanup on mid-stream halt.** Two `Stream.resource` enums with `Agent`-tagged after-funs, zipped and halted mid-stream via the suspend protocol → `{:halted, [0]}` and **both** after-funs fired (`[:a, :b]`), confirming `do_zip_close/1` runs on the `{:halt, acc}` re-entry path.

Reading neighbour modules was not the instrument that closes the hedge here: the only observable axis is which acc-shapes the two implementations disagree on, which is visible **only** by pushing inputs through them — done above. The `Enumerable.reduce`/`Stream` machinery itself is treated as the trusted protocol contract (scoping boundary, not an unverified assumption about this file); nothing in this file's behavior was left asserted.

### Dropped in re-review

None. All first-pass conclusions ("No findings" plus the five ruled-out entries) are preserved, each re-confirmed by fresh execution this pass. No dismissal was moved toward greater confidence without executed evidence, and no finding was silently dropped.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`chunk_every` overlap (`step < count`) retains the wrong elements** via `Enum.take(acc_buffer, acc_count - step)` on the reversed buffer. Ruled out: `Enum.chunk_every(1..10, 3, 2)` → `[[1,2,3],[3,4,5],[5,6,7],[7,8,9],[9,10]]`, correct overlap.
  - Basis: **verified (executed)**. Axes examined: overlap step, buffer-reversal order. | Not examined: none.
