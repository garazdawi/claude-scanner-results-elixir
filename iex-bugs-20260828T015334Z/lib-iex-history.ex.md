# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/history.ex
<!-- sha256: 737ba873c95bb6651b86568551f3f2aa9dba2fff3497e635b448f57b8a08ec5c -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:08:16Z_

I've read the whole file and exercised every documented public path (`init/1`, `append/3`, `each/2`, `nth/2`) plus the internal `prune`/`has_bin` GC-detection machinery across many value shapes. Everything behaves correctly.

## Summary
After exercising all documented public functions and the pruning/binary-detection internals against boundary and adversarial inputs, I found no triggerable correctness bugs in `IEx.History`.

## Bugfixes

No findings

### Ruled out

- **`nth/2` positive/negative index off-by-one** — The positive clause guard `n - start >= 0 and n - start < size` maps history number `n` to queue index `n - start` over exactly `start..start+size-1`; the negative clause `n < 0 and size + n >= 0` with `get_nth(:queue.reverse(q), abs(n) - 1)` maps `-1..-size` to the newest..oldest. *verified*: with a 3-entry history `nth 1/2/3` returned `:a/:b/:c` and `nth -1/-2/-3` returned `:c/:b/:a`; after pruning to `start=3`, `nth(3)`→3 and `nth(-1)`→5. Both boundaries are inclusive and correct.

- **`nth/2` out-of-bounds crash contract** — Clauses 3/4 deliberately `raise` with an informative message; this matches the intended contract for `v/1` (a bad index tells the caller the preserved range). *verified*: `nth(s,2)` on a pruned history raised `"v(2) is out of bounds, the currently preserved history ranges from 3 to 5 ..."`, and the empty-history case raised the "no entries were stored" message. Raising here is the contract, not a bug.

- **`prune/4` accumulator/`start` threading** — `counter` starts at `start` and increments once per popped entry; the base clause sets `start` to the final `counter`, keeping `start` equal to the oldest surviving entry's number, and `collect?` is OR-accumulated across all pruned entries. *verified*: appending 1..5 with `limit=3` yielded `{start:3, size:3, queue:[3,4,5]}`; negative limit left `{start:1, size:10}` (unlimited, nothing pruned, `start` correctly untouched).

- **`has_bin`/`has_binary` crash-on-arbitrary-value** — The `when byte_size(val) > 64` guard cannot raise (guard-context exceptions become guard failure, falling through to the `has_bin(_) -> false` clause); tuples/lists recurse, and `throw(:found)` is the only throw, caught by `catch :throw, :found`. *verified*: forcing the prune path (`limit=0`) over big binaries, nested tuples/lists, improper lists, maps, pids, refs, funs, floats, atoms, empty tuple/list, and a sub-byte bitstring produced "OK" for every case with no crash or uncaught throw.

- **`has_bin` >64-byte threshold** — Matches the ERTS 64-byte heap/refc-binary boundary, so only off-heap binaries trigger `garbage_collect`; a 64-byte heap binary intentionally does not. Correct by design, not an off-by-one.

- **`each/2` ordering** — `to_list/1` uses `:queue.to_list`, which yields front-to-back; since `append` uses `:queue.in` (enqueue at back), the front is the oldest, matching the doc "starting from the oldest one." Consistent with the verified `nth` ordering above.

## Risks / caveats
- `@moduledoc false` marks the module internal, but its functions carry `@doc` strings and are reached via documented IEx helpers (`v/1`, history display), so I treated `init/1`, `append/3`, `each/2`, `nth/2` as the public surface per the review's "documented public functions" scope.
- `has_bin/1` intentionally does not descend into maps or other containers (only lists/tuples, as its comment states); a large binary buried inside a map value will not trigger the GC hint. This is a documented limitation of a best-effort memory optimization, not a correctness defect, so I did not report it.
- I did not exercise `nth/2` with non-integer `n` or a negative `start`, as both would require the caller to violate the function's implicit integer/positive-counter contract (out of scope).

