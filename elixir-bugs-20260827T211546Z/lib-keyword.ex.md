# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/keyword.ex
<!-- sha256: ea0fa686281e2c30e34b03b3475e7e2ffbefcebbb22324eda8cd29d072a79cf7 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/keyword.ex
<!-- sha256: ea0fa686281e2c30e34b03b3475e7e2ffbefcebbb22324eda8cd29d072a79cf7 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (toolchain: Elixir 1.21.0-dev / Erlang/OTP 29, built from this repo). The keyword.ex source is not present in this sandbox checkout; all behavioural claims below were re-executed against the installed, same-version compiled `Keyword` module and reproduced exactly. Line numbers are carried from the first pass/repo tree._

The first-pass reviewer flagged the hedge phrase **"did not exhaustively"** (the declined fuzz of the `validate/2`/`validate_merge`/`validate_fallback` state machine). This pass discharges it: the fuzz was re-run with a fresh seed (`{7,8,9}`) and an independently-written reference, 50,000 cases, **0 semantic mismatches**. No neighbour-module assumption remained to verify — `keyword.ex` is self-contained; its only cross-module dependence is on `:lists`/`Enum`/`List` BIFs, whose behaviour is exercised directly by the executed checks below rather than assumed. Both findings and all three ruled-out entries are preserved. No finding is dropped.

## Summary
`Keyword.get_and_update!/3` raises a `KeyError` whose `:term` (and error message) shows the keyword list **reversed**, unlike every sibling raising function. Separately, `validate/2` / `validate!/2` emit **duplicated key atoms** in their error list / message when an unknown key (or a key present ≥3 times) recurs — and, as the refreshed fuzz surfaced, they also **mix unknown and duplicate keys into a single flat error list**. Both are diagnostic-only; the module's return values on valid input are correct (fuzz-confirmed).

## Bugfixes

#### `CSCAN-keyword-get-and-update-bang-reversed-term`
- **Finding ID:** `CSCAN-keyword-get-and-update-bang-reversed-term`
- **Severity:** Low
- **Confidence:** high
- **Location:** `keyword.ex:597-599`, `get_and_update!/4` empty-list clause (`raise KeyError, key: key, term: acc`)
- **Issue:** When `key` is absent, the recursion has consumed the whole list into the reverse accumulator `acc`, so `defp get_and_update!([], key, _fun, acc) -> raise KeyError, key: key, term: acc` reports `acc`, which is the input list in **reverse order**. Every other raising function in the module (`fetch!/2:640`, `replace!/4:934-935`, `update!/4:1191-1192`, `pop!/2:1396`) threads the *original* list and reports it in input order; `get_and_update!/3` uniquely reuses the reverse accumulator instead of threading the original.
- **Class:** Contract violation (misleading error data / documentation-as-contract)
- **Impact:** A caller who hits a missing key gets a `KeyError` message listing the pairs in reversed order, and any code rescuing the exception and inspecting `e.term` receives a reversed list. Purely diagnostic — no wrong return value or crash on valid input — but actively misleads debugging and is inconsistent with the rest of the module.
- **Evidence:** `verified` (re-run this pass).
  - `Keyword.get_and_update!([a: 1, b: 2, c: 3], :z, fn _ -> {1,1} end)` → `e.term` is `[c: 3, b: 2, a: 1]`; `Exception.message/1` prints `key :z not found in:\n\n    [c: 3, b: 2, a: 1]`.
  - Contrast: `Keyword.fetch!([a: 1, b: 2, c: 3], :z)` → `e.term` is `[a: 1, b: 2, c: 3]` (correct order).
- **Suggested fix:** Thread the original list through, mirroring `update!/4`: seed the public `get_and_update!/3` (line 576) with `get_and_update!(keywords, key, fun, [], keywords)`, add the arg to the recursive clauses, and raise with `term: original`. Minimal alternative: raise with `term: :lists.reverse(acc)` in the empty clause.

#### `CSCAN-keyword-validate-duplicate-error-keys` — Upgraded from hedged assumption in first pass
- **Finding ID:** `CSCAN-keyword-validate-duplicate-error-keys`
- **Severity:** Low
- **Confidence:** low
- **Location:** `keyword.ex:307-318` (`validate_fallback/3`, the `[key | bad_keys]` accumulation) surfacing in `validate/2:274-276` and `validate!/2:388-397`
- **Issue:** `validate_fallback/3` appends `key` to `bad_keys` on every occurrence of a bad key without de-duplicating. When a single unknown key appears more than once, or an in-`allowed` key appears **three or more** times (first occurrence consumed by `find_key!`, each later one fails and is re-accumulated), the returned `{:error, list}` contains the atom multiple times. Additionally — surfaced by the refreshed fuzz — when a list contains *both* unknown keys and duplicated allowed keys, the single flat error list mixes both categories (e.g. `{:error, [:c, :b]}` where `:c` is a duplicated allowed key and `:b` is unknown). `validate!/2` passes that list straight into its message, so the raised text reads `unknown keys [:a, :a] ...` or `duplicate keys [:a, :a] ...`.
- **Class:** Contract violation (misleading error data). The public doctests (`keyword.ex:260-266`, `366-372`) only ever show each bad key **once**, so the duplicated/mixed form is an undocumented, awkward-reading variant of the documented contract.
- **Impact:** Diagnostic only — the `{:ok, ...}` path and the *set* of rejected keys are correct (fuzz-confirmed); just the presentation repeats atoms and blends categories. No wrong return value on valid input, no crash. Low confidence because the error-list ordering/multiplicity is explicitly undocumented ("no guarantees"), so this may be judged intended looseness rather than a defect.
- **Evidence:** `verified` (executed this pass).
  - `Keyword.validate([a: 1, a: 2], [:c])` → `{:error, [:a, :a]}`; `Keyword.validate!([a: 1, a: 2], [:c])` → `unknown keys [:a, :a] in [a: 1, a: 2], the allowed keys are: [:c]`.
  - `Keyword.validate([a: 1, a: 2, a: 3], [:a])` → `{:error, [:a, :a]}`; `validate!` → `duplicate keys [:a, :a] in [a: 1, a: 2, a: 3]`.
  - Mixed category (from fuzz): input `[d,b,c,a,c]` against allowed `[:c,:a,:d]` → `{:error, [:c, :b]}` — duplicated allowed key `:c` and unknown key `:b` in one flat list.
  - Contrast (documented, single occurrence): `Keyword.validate([a: 1, a: 2], [:a])` → `{:error, [:a]}` → `duplicate keys [:a] in [a: 1, a: 2]`.
- **Suggested fix:** `Enum.uniq` the accumulated `bad` list in `validate_fallback/3:318` before returning, or de-duplicate `invalid_keys` in `validate!/2` before building the message.

## Risks / caveats
- The installed toolchain is Elixir 1.21.0-dev built from this same repo/version; the earlier pass recorded all 86 module doctests passing, ruling out disagreement in the *documented* examples. (Doctests were not re-executed this pass — the source file is not checked out in this sandbox — but every behavioural claim was re-run directly against the compiled module.)
- **Hedge discharged with executed evidence (re-confirmed this pass):** the first pass declined to fuzz the `validate/2` state machine. This pass ran a **fresh** 50,000-case randomized differential test (`/tmp/vfuzz2.exs`, seed `{7,8,9}`, independent from the first pass): random keyword inputs of 0–5 entries over key alphabet `{:a,:b,:c,:d}` (including duplicates and unknowns) against `values` lists of 1–4 unique allowed keys (mixed bare-atom / `{key, default}` forms), compared against an independently-written reference. Comparing the **semantic decision** (`:ok` vs `:error`) and, on the `:ok` path, the full result **set** (defaults overlaid with provided values): **0 mismatches**. A first, stricter run (`/tmp/vfuzz.exs`) that compared exact error-list contents flagged only the ordering/multiplicity/category-mixing already captured in `CSCAN-keyword-validate-duplicate-error-keys` — no decision or result-set divergence. Residual gap: fuzzing bounds coverage, not proves absence — keys beyond a 4-atom alphabet, lists longer than 5, and `values` lists with duplicate keys (a spec violation) were not swept.

### Ruled out
- **`merge/2` skips second-argument validation when the first list is empty** — `Keyword.merge([], [1,2,3])` returns `[1,2,3]` (line 1056 short-circuits before `keyword?/1`), whereas `Keyword.merge([a: 1], [1,2,3])` raises `ArgumentError` (`expected a keyword list as the second argument, got: [1, 2, 3]`). Basis: `verified` (ran both this pass). Axes examined: input shape of the second argument (valid keyword vs. non-keyword list) with empty vs. non-empty first list — the exact axis that distinguishes the short-circuit. Not examined: none. Not a primary finding: triggering it requires passing a non-keyword list as the second argument, violating the `@spec merge(t, t)` contract; the empty-first-list shortcut is a genuine but out-of-scope inconsistency.
- **`equal?/2` treats duplicate-key lists with different first-value semantics as equal** — Basis: `verified` (ran this pass). `Keyword.equal?([a: 1, a: 3], [a: 3, a: 1])` → `true`, even though `Keyword.get/2` returns `1` vs `3` respectively (first-value differs). Axes examined: duplicate-key ordering under `:lists.sort/1`, and the observable first-value divergence via `get/2`. Not examined: none for this claim. Not a defect: this is the long-standing documented definition ("same keys and those keys contain the same values") and matches the passing doctest at `keyword.ex:987`.
- **`delete_first_key/2` lacks a `[]` clause** (`keyword.ex:793-799`). Basis: `asserted` — `delete_first/2` (line 786) only calls `delete_first_key/2` after `:lists.keymember(key, 1, keywords)` returns `true` (line 787-788), guaranteeing the key is present before the list is exhausted, so the missing clause is unreachable. Axes examined: none executed. Not examined: forcing `delete_first_key/2` to reach `[]` — which is unreachable through the public API because the `keymember` guard is a hard precondition, so no input carries that axis. I attempted to identify a public-API input that drives the private recursion to `[]`; none exists (the guard is the precondition), so the distinguishing axis cannot be pushed through the implementation. Entry stays `asserted` (per the re-review rule: reading a neighbour or re-reasoning does not clear an untried axis).

### Did I verify this?
- `CSCAN-keyword-get-and-update-bang-reversed-term`: **yes** — re-ran, `e.term` = `[c: 3, b: 2, a: 1]` vs `fetch!` `[a: 1, b: 2, c: 3]`. High confidence.
- `CSCAN-keyword-validate-duplicate-error-keys`: **yes** — executed both `validate/2` and `validate!/2` on the duplicating and mixed-category inputs; observed `[:a, :a]` and `[:c, :b]` in list and message. Low confidence on *severity* (undocumented error-list order/multiplicity), not on the observation.
- `validate/2` state-machine correctness: **yes** — fresh 50,000-case differential fuzz (independent seed + reference), 0 semantic/result-set mismatches (see caveat). Coverage-bounded, not exhaustive.
- `merge/2` empty-first shortcut: **yes** — ran both branches this pass.
- `equal?/2` duplicate semantics: **yes** — ran, `true` with divergent first values.
- `delete_first_key/2` unreachable `[]`: **no execution** — guarded by an unreachable-by-construction precondition; no public-API input carries the axis, so it stays `asserted`.

### Dropped in re-review
None. Every finding and ruled-out entry from the first pass is preserved. The `validate`-machine hedge flagged by the first-pass reviewer ("did not exhaustively") was re-discharged with a fresh independent fuzz (0 semantic mismatches) and expanded `CSCAN-keyword-validate-duplicate-error-keys` with the newly-observed unknown/duplicate error-list mixing. The two previously-converted ruled-out entries (`merge/2`, `equal?/2`) remain `verified`; the `delete_first_key/2` entry remains `asserted` because its distinguishing axis is unreachable through the public API and therefore cannot be executed.
