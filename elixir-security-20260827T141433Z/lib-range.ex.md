# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/range.ex
<!-- sha256: c0b159472f6c7e833fae5c05ca4662a78cdb982f98150a3a661392eee08caa68 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:22:05Z_

I have thorough verification now. Let me consolidate my findings.

I've confirmed via execution:
- `disjoint?/2` is correct across an exhaustive small search (endpoints −12..12, steps ±1..3) **and** 300k randomized large-magnitude cases (exercising the extended-GCD alignment path) — **0 mismatches**.
- `size/1` correct exhaustively (**0 mismatches**); `split/2` satisfies all documented invariants exhaustively (**0 failures**).
- The public constructor `new/3` rejects a zero step with `ArgumentError`.
- The only anomalies (nil-step `size` returning `0`/raising) require hand-building a struct via `struct/2`, bypassing `@enforce_keys` — an explicit violation of the documented "do not create ranges by hand" contract.

Here is the report.

---

## Summary

`Range` is a pure integer-arithmetic module with no atom/code/file/network/deserialization/Unicode sinks; its non-trivial predicates (`disjoint?/2`, `size/1`, `split/2`) were verified correct by exhaustive and randomized testing, and no security issue is reachable through its documented public API with valid inputs.

## Security

No findings.

### Ruled out

- **`to_list/1` unbounded list materialization (`:lists.seq/3`, range.ex:424).** Resource-consumption *shape* (a tiny 3-integer range yields an O(n) list, so `Range.to_list(0..10_000_000_000)` can OOM the node). Ruled out as a file defect: materializing the whole range is the entire documented purpose of `to_list/1` ("Converts a range to a list"), identical in kind to `Enum.to_list/1`; there is no safer in-module replacement, and the obligation to bound an untrusted range belongs to the caller, not this function. Surfaced under Risks below. Basis: verified the semantics; the DoS is inherent, not a bug.
- **Nil-step range misbehavior in `size/1` / `to_list/1` (range.ex:259–261, 421–425).** `Range.size(struct(Range, first: 5, last: 1, step: nil))` returns `0` (should be 5) and `Range.size(struct(Range, first: 1, last: 5, step: nil))` raises `ArithmeticError` (`div/2` by `nil`). Verified by running. Ruled out: a `nil` step violates the `@type step :: pos_integer | neg_integer` / `t` contract and is only obtainable by hand-constructing the struct via `struct/2` (bypassing `@enforce_keys`), which the moduledoc explicitly forbids ("you should not modify nor create ranges by hand"). The documented constructors `new/2`, `new/3`, and `..`/`..//` all set a non-zero integer step — verified `new(1,5,0)` raises `ArgumentError`. Out of scope per the contract-violation exclusion.
- **Legacy `# TODO: Remove me on v2.0` fallback clauses (range.ex:264–267, 432–435) as dead defensive code.** These match pre-1.12 *stepless* maps (no `:step` key at all). Verified reachable and correct: `size(%{__struct__: Range, first: 1, last: 5})` → `5`, `to_list(...)` → `[1,2,3,4,5]`. They are shadowed only for maps that *do* carry a `step` key (handled by the earlier `%Range{}` clauses), so they are not dead for their intended pre-1.12-term target. Not a finding.
- **`disjoint?/2` wrong-answer (Validation class), range.ex:479–514.** A wrong result could matter if used as an overlap/safety gate. Ruled out by verification: 0 mismatches against a MapSet-intersection oracle over an exhaustive small grid plus 300k randomized cases with magnitudes to ±200 and steps to ±7 (exercising `Integer.extended_gcd`/`floor_div`). All arithmetic uses BEAM arbitrary-precision integers, so no overflow path exists.
- **Zero-step crash / division-by-zero in `normalize/3`, `size/1` clause 3, `split/4`.** All require a `step: 0` range, which `new/3` rejects (verified `ArgumentError`) and the `..//` operator rejects; obtainable only by contract-violating hand construction. Out of scope.
- **`shift/2` (range.ex:289–293) integer overflow.** None possible — BEAM integers are unbounded; `shift` is O(1) arithmetic on caller-supplied integers.

## Risks / caveats

- **Residual (caller-side, not a defect in this file):** `Range.to_list/1` and `Range.size/1`→`to_list` amplify a constant-size range into an O(n) allocation. An application that builds a range from **untrusted bounds** (e.g. `first`/`last` from an HTTP request) and then materializes it (`Range.to_list/1`, `Enum.to_list/1`, or any eager `Enum` over the range) can be driven to memory/CPU exhaustion. Bound the span before materializing; ranges are lazy, so prefer streaming/`Enum.count`/`Range.size` over `to_list` on attacker-influenced ranges.
- **Assumptions:** I treated only valid `%Range{}` values (integer `first`/`last`, non-zero integer `step`) as in-scope inputs, consistent with the module's type specs and the explicit "do not create ranges by hand" contract; behavior under hand-forged nil/zero-step structs was examined and shown to require that contract violation.
- **Coverage:** correctness was established by exhaustive small-domain and large randomized testing, not formal proof; the tested magnitude/step ranges are wide but finite. Given the pure arbitrary-precision-integer implementation with no unbounded loops, no additional attack surface (atoms, code-eval, I/O, dispatch, Unicode, crypto) exists in this file.

