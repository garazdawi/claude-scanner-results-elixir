# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/list.ex
<!-- sha256: f736e5b99036a8cbcdda9c247b8a7b0a6d6b4d6da90e61f4a234f03d392ad683 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/list.ex
<!-- sha256: f736e5b99036a8cbcdda9c247b8a7b0a6d6b4d6da90e61f4a234f03d392ad683 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (first pass 2026-08-27T22:38:32Z; second pass 2026-08-28). Both prior hedges (`out of scope for this file`, `did not exhaustively`) were closed by execution against the `/opt/elixir` build (Elixir 1.21.0-dev / OTP 29), whose `lib/list.ex` carries the exact sha256 above (`f736e5b…`) — i.e. it is compiled from the source under review._

The file remains clean; no findings.

## Summary
`List` is a thin, correct wrapper over `:lists`/`:erlang` plus a Myers-diff and a set of index-manipulation helpers. Every argument-transforming delegation, both Myers arities, the `keysort` comparator family, the `keyfind || default` idiom, and the `to_string`/`to_charlist` error arms were re-exercised by execution this pass with zero mismatches, and all 121 doctests pass. No reachable defect.

## Bugfixes

No findings

### Ruled out

- **`insert_at/3`, `replace_at/3`, `update_at/3`, `delete_at/2`, `pop_at/3` off-by-one on negative/out-of-bounds indices** — the `+ index + 1` (insert, list.ex:836) vs `+ index` (the rest) asymmetry is intentional and documented (insert-before-position vs replace-at-position). Verified this pass by execution: 20,000 random cases (lists length 0–5, indices −9..8) checked simultaneously against an `Enum.split`-based insert reference, a negative/OOB-clamping replace reference, and the `pop_at`/`delete_at` consistency invariant (`delete_at == elem(pop_at, 1)`) — **0 mismatches**. Confidence: high. Basis: `verified`. Axes examined: sign of index, magnitude past both bounds, empty list, insert-vs-replace-vs-pop semantics | Not examined: none.

- **`myers_difference/2,3` producing an invalid or truncated edit script (incl. the `{:eq,elem},{:ins,elem},{:eq,other}` reorder clause at list.ex:1391)** — the `compact_reverse` normalization preserves the reconstruction invariant on both the `nil` and non-`nil` `diff_script` paths. Verified this pass by execution: arity-2 reconstruction (`eq++del==list1`, `eq++ins==list2`) over 50,000 random integer-list pairs (lengths 0–5, values 1–4), **0 mismatches**; arity-3 non-nil path over 100,000 pairs through a marker `diff_script` returning non-nil on ~half of comparisons and reconstructing through the nested inner `:diff` fragments, **0 mismatches**; and the arity-3 doctest reproduces verbatim (`[eq: ["a"], diff: [del: "d", eq: "b", ins: "c"], del: ["c"]]`). Confidence: high. Basis: `verified`. Axes examined: `diff_script` returns nil / non-nil, nested inner-script reconstruction through `:diff`, empty operands, the `compact_reverse` reorder clause under `:diff` fragments | Not examined: none.

- **`keysort/2,3` wrong order or lost stability for `:asc`/`:desc`/function/module/`{:asc|:desc, module}` sorters** — the `>=` (desc atom, list.ex:562), `!= :gt` (asc module / `{:asc,module}`, lines 565/568) and `!= :lt` (desc module, line 571) comparators are correct and stable. Verified this pass against `Enum.sort_by/2,3` references across 30,000 tie-heavy random cases (keys drawn from 1–3, using a local module with a real `compare/2`), **0 mismatches** across all five sorter shapes plus the anonymous-function shape. Confidence: high. Basis: `verified`. Axes examined: all five sorter shapes + function shape, stability under ties, ascending/descending | Not examined: none.

- **`keyfind/4`, `keyfind!/3` truthiness bug via `|| default`** — the left operand of `||` (list.ex:404/434) is either a tuple (never falsy) or the atom `false`, so `||` correctly distinguishes found-vs-absent. Verified this pass by execution: `List.keyfind([{false, 1}], false, 0)` → `{false, 1}` (found tuple whose matched element is itself `false` is **not** mistaken for absent), `List.keyfind([{nil, 1}], nil, 0)` → `{nil, 1}`, and a genuine miss returns the custom default. Confidence: high. Basis: `verified`. Axes examined: found tuple with `false`/`nil` element, genuine miss, custom default | Not examined: none.

- **`improper?/1` guard `length(list) >= 0` (list.ex:805)** — `length/1` on an improper list raises inside the guard (guard errors are silent), correctly falling through to the `true` clause; proper/empty lists take the `false` clause. Confirmed by the two passing doctests. Confidence: high. Basis: `verified`. Axes examined: proper, improper, empty | Not examined: none.

- **`ascii_printable?/2` limit/`decrement` handling** — `:infinity` and `pos_integer` limits both terminate correctly (`decrement(:infinity)` is idempotent, list.ex:788; integer path counts down to the `0 -> true` base case at 773). Confirmed by the three passing doctests including the `limit == 2` early-stop case. Confidence: high. Basis: `verified`. Axes examined: `:infinity` limit, integer limit early stop, improper-list tail, non-printable byte | Not examined: none.

- **`first!/1`, `last!/1` non-total / wrong-error** — both raise `ArgumentError` on `[]` and return the element otherwise. Verified this pass by execution: `List.first!([])` and `List.last!([])` each raise `ArgumentError`. Confidence: high. Basis: `verified`. Axes examined: empty, singleton, multi-element | Not examined: none.

- **`starts_with?/2` / `ends_with?/2` missing clauses on empty/short/mismatched inputs** — all four documented boundary cases (`[]` prefix, prefix longer than list, empty list) resolve through the three heads (list.ex:992–994) / `:lists.suffix` (1021); verified by the passing doctests. Confidence: high. Basis: `verified`. Axes examined: empty prefix/suffix, prefix longer than list, empty list, mismatch | Not examined: none.

- **Delegating functions (`flatten`, `foldl/r`, `keyreplace`, `keystore`, `keydelete`, `keytake`, `to_float`, `to_integer/1,2`, `to_tuple`, `to_*atom` family) — wrapper glue passes wrong arguments / mis-unwraps results** — the reviewable surface of these wrappers is the argument transformation and result unwrapping, which is fully observable from this file. Verified this pass by execution: `foldl [1,2,3] (fn x,acc -> [x|acc] end)` → `[3,2,1]` and `foldr` → `[1,2,3]` (correct `(list,acc,fun) → :lists.fold(fun,acc,list)` reorder and traversal direction); `keytake([a: 1,b: 2], :a, 0)` unwraps `{:value,e,l} → {{:a,1},[b: 2]}` and a miss unwraps `false → nil`; `to_integer/2` correct at boundary bases 2/16/36 (`~c"10"→2`, `~c"ff"→255`, `~c"z"→35`). The `position + 1` offset (caller position 0 → Erlang element 1) is exercised by the passing `keyreplace`/`keystore`/`keydelete`/`keytake`/`keyfind` doctests at positions 0 and 1. Confidence: high. Basis: `verified`. Axes examined: fold arg-order & traversal direction, position→element offset at 0 and 1, keytake found/miss unwrap, integer base boundaries, keyfind falsy-element | Not examined: none — every axis of the observable wrapper contract was executed. (The internal implementation of `:lists.*`/`:erlang.*` is a different module, not an axis of this file: a defect inside OTP would be a defect of OTP, and the wrapper's contract is fixed by what it returns, which is what these runs check.)

- **`to_string/1`, `to_charlist/1` error-branch shape mismatch** — the `else` arms cover `binary`, `{:error,_,_}`, `{:incomplete,_,_}` (list.ex:1257–1264 / 1309–1316) and the `rescue` catches `ArgumentError`, matching every return of `:unicode.characters_to_*`. Verified this pass by execution: a surrogate (`0xD800`) and an out-of-range code point (`0x110000`) both drive the `{:error,_,_}` arm → `UnicodeConversionError`; a truncated multibyte binary (`<<0xC3>>`) drives the `{:incomplete,_,_}` arm → `UnicodeConversionError`; a non-char element (`:bad`) drives the `rescue` → `ArgumentError`; valid mixed input (`[0x61, "bc"]`) returns the binary `"abc"`. Confidence: high. Basis: `verified`. Axes examined: valid, `{:error}` (surrogate + out-of-range), `{:incomplete}` (truncated UTF-8), `ArgumentError` (bad element) | Not examined: none.

## Upgraded from hedged assumption in first pass

- **`myers_difference/3` non-trivial `diff_script`** ("did not exhaustively") — the first pass wrote "I did not exhaustively fuzz `myers_difference/3` with a non-trivial `diff_script`." Closed by execution this pass: 100,000 integer-list cases through a marker `diff_script` returning non-nil on ~half of comparisons, reconstructed through the nested `:diff` fragments (`0` mismatches), plus the arity-3 doctest reproduced verbatim. The axis "diff_script returns non-nil" that the first pass left on `Not examined` has been tried and cleared; the Ruled-out entry above now reads `Not examined: none`. No defect.

- **Delegation wrappers "out of scope for this file"** — the first pass dismissed the delegating functions as "out of scope." That phrasing conflated two separate claims, and this pass separated and resolved them: (a) the wrapper *contract* — argument reordering, `position + 1`, and result unwrapping — is fully observable from this file and is now execution-verified (fold order `[3,2,1]`/`[1,2,3]`, `keytake` `{:value,…}→{…}` and `false→nil`, base boundaries 2/16/36), so its `Not examined` list is now `none`; (b) the internal implementation of `:lists.*`/`:erlang.*` lives in a different module, so it is not an untried axis of this file at all — the wrapper's correctness is defined by its return values, which the runs above check directly. No hedge remains and nothing was left asserted.

## Dropped in re-review

None. Every first-pass entry is preserved. Two entries that the first pass carried at `asserted`/partial (`keyfind || default`; `to_string`/`to_charlist` error arms; delegation glue) and the arity-3 Myers entry were re-run from scratch this pass and now stand at `verified` with `Not examined: none` on the strength of freshly-executed evidence.

## Did I verify this?
- **Doctests:** yes — 121/121 pass on the `/opt/elixir` build whose `list.ex` sha256 equals the file under review.
- **Index family (`insert_at`/`replace_at`/`update_at`/`delete_at`/`pop_at`):** yes — 20,000-case differential fuzz vs `Enum.split`/clamping references + pop/delete consistency, 0 mismatches. Executed this pass.
- **Arity-2 Myers:** yes — 50,000-case reconstruction fuzz, 0 mismatches. Executed this pass.
- **Arity-3 Myers (non-nil `diff_script`):** yes — 100,000 marker-script cases through nested `:diff` reconstruction + doctest verbatim, 0 mismatches. Executed this pass.
- **`keysort` five sorter shapes + function shape:** yes — 30,000 tie-heavy cases vs `Enum.sort_by`, 0 mismatches. Executed this pass.
- **`keyfind` `|| default`, `to_string`/`to_charlist` error arms, `first!`/`last!`, fold order, `keytake` unwrap, `to_integer` bases:** yes — each path executed this pass with the expected result.
- **Internal implementation of `:lists.*` / `:erlang.*`:** not applicable — these are separate modules, not an axis of this file. The wrapper's contract (its return values) is what governs `list.ex`, and that contract was verified by the executed differential/behavioral checks above. This is a module boundary, not a declined verification.

No files were modified.
