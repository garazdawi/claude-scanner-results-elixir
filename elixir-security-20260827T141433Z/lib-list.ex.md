# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/list.ex
<!-- sha256: f736e5b99036a8cbcdda9c247b8a7b0a6d6b4d6da90e61f4a234f03d392ad683 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/list.ex
<!-- sha256: f736e5b99036a8cbcdda9c247b8a7b0a6d6b4d6da90e61f4a234f03d392ad683 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T (hedge-resolution pass)_

Two first-pass hedges were flagged: "I did not measure peak RSS" (myers memory) and "the cost is in `:erlang.list_to_integer` (out of scope for this file)" (`to_integer` scaling). Both were resolvable by execution in the sandbox rather than by reading neighbour modules, because both bottom out in Erlang BIFs, not in sibling `src/` modules. I ran both. The myers memory claim is upgraded from `asserted` to `verified` (instrumented process memory). The `to_integer` claim is upgraded to `verified` with measured scaling and the "out of scope" framing removed; it remains a ruled-out non-finding because the amplification is weak and input-proportional.

## 1. Summary
`List` is almost entirely pure list manipulation; the only security-relevant behavior is an asymmetric resource-consumption blowup in `myers_difference/2,3` — now confirmed to explode in **both** CPU and heap (4.3 GB resident term memory for 16k total elements). The atom- and Unicode-conversion sinks are either documented dangerous primitives with safe alternatives or correctly surface their errors.

## 2. Security

#### `CSCAN-list-myers-difference-quadratic`
- **Finding ID:** `CSCAN-list-myers-difference-quadratic`
- **Severity:** Low
- **Confidence:** high (behavior — CPU **and** memory now both instrumented) / medium (that it rises to a defect vs. inherent cost)
- **Location:** `list.ex:1343` / `1364` (`myers_difference/2,3`), driver `find_script`/`each_diagonal` at `list.ex:1369-1474`
- **Class:** Resource consumption (CWE-770 / CWE-400)
- **Issue:** The Myers diff keeps, for every diagonal path, its full accumulated edit list (`move_right`/`move_down` prepend to `edits` at `list.ex:1430-1461`; `find_script` at `list.ex:1374-1379` grows the envelope until edit-distance `D` is reached). Cost is at least O(N·D) time and O(D²) retained path memory. For two lists with no elements in common, `D ≈ 2N`, so both explode with input size. There is no cap or short-circuit on input length or on `D`.
- **Impact / threat:** An application that diffs two *externally supplied* lists through the documented public API (`List.myers_difference/2`) can be driven to burn tens of seconds of CPU **and multiple gigabytes of heap** from a small input. The crossed boundary is the caller forwarding attacker-controlled list content into a diff (diffing user-submitted content, two files' line-lists, two API payloads). The `@doc` at `list.ex:1320-1340` advertises only the paper's "O(ND)" and gives no memory warning.
- **Evidence:** `verified` — both time and memory instrumented this pass.
  ```
  # CPU (wall-clock), original first-pass run:
  n=1000  216 ms | n=2000 1026 ms | n=4000 5859 ms | n=8000 37854 ms

  # This pass — same workload, diffing in a spawned process, reading
  # Process.info(:memory) and :heap_size after the call returns:
  n=1000  225.0 ms  proc_mem=79003.5 KB    heap_words=1199557
  n=2000  864.7 ms  proc_mem=283081.7 KB   heap_words=4298223
  n=4000  7027.5 ms proc_mem=1217197.1 KB  heap_words=18481566
  n=8000 29224.4 ms proc_mem=4361435.3 KB  heap_words=66222786
  ```
  Memory grows ~3.6×–4.3× per input doubling (super-quadratic), matching the O(D²)-retained-path reading. **16k total elements (~128 KB of term data in → 4.3 GB resident):** the asymmetry factor is ~34,000×. This directly confirms the memory half of the finding that the first pass could only assert.
- **Suggested fix:** Smallest honest change is documentation + a caller-side guard: note in the `@doc` that time **and memory** grow with the edit distance and that untrusted, dissimilar inputs should be length-bounded before diffing. If a code change is wanted, add an optional `max_edits`/length bound that returns a coarse `[del: list1, ins: list2]` (or raises) once the envelope exceeds a caller-supplied threshold.

### Ruled out
- **`to_atom/1` (`list.ex:1025`) and `to_unsafe_atom/1` (`list.ex:1060`) — atom-table exhaustion (CWE-770).** Basis: read + `verified` these are thin wrappers over `:erlang.list_to_atom/1`. Creating an atom is the *documented purpose* of both: `to_atom/1` is `@deprecated` pointing to `to_existing_atom/1`/`to_unsafe_atom/1`, and `to_unsafe_atom/1` carries a prominent "Dynamic Atom Creation" warning naming untrusted socket/web input. Exploiting them requires the victim to pass attacker data into a primitive explicitly documented as unsafe — analogous to `Code.eval_string`; not a defect in this file. Safe forms (`to_existing_atom/1,2`) are present.
- **`to_existing_atom/1` (`list.ex:1095`) and `to_existing_atom/2` (`list.ex:1119`).** Basis: `verified`. Use `:erlang.list_to_existing_atom/1` (never creates atoms — `List.to_existing_atom(~c"zzz_never_seen_atom_9182")` raised `ArgumentError`), and `/2` additionally rejects anything outside the caller's allow-list. These are the safe replacements, not sinks.
- **`keysort/3` module dispatch (`list.ex:554-571`).** `keysort_fun(module, …)` calls `module.compare/2` on a caller-supplied atom. Basis: read. No atom is *created* here (the module atom is passed as a term), and the comparison module is chosen by the caller per the documented `sorter` contract; a caller forwarding an untrusted module is opting into dispatch, same pattern excluded for `to_atom`.
- **`duplicate/2` (`list.ex:190`).** Allocates O(n) from an explicit integer count; cost is proportional to the number the caller asked for — not an asymmetric amplification.
- **`to_integer/1,2` (`list.ex:1168,1186`), `to_float/1` (`list.ex:1152`) — no meaningful asymmetric DoS.** Basis: `verified` — measured this pass (removing the first-pass "out of scope" hedge). `List.to_integer` on all-`9` charlists: 100k digits → 3.0 ms, 200k → 8.8 ms, 400k → 27.9 ms, 800k → 73.6 ms. Growth is ~2.6–3.2× per input doubling (exponent ≈ 1.5, mildly super-linear), and the cost is bounded by an input that must be *as large as the work* — 800 KB of source digits buys only ~74 ms. To reach a one-second stall the attacker must ship several MB of literal digits, an input-proportional cost with no amplification worth a finding. The super-linear term lives inside `:erlang.list_to_integer/1` (a BIF), but that does not put it out of scope — I measured `List.to_integer`'s end-to-end behavior directly and it does not clear the bar for a resource-consumption finding.
- **`to_string/1` (`list.ex:1237`), `to_charlist/1` (`list.ex:1290`).** Basis: `verified`. Correctly surface conversion failures: surrogate `0xD800` and `> 0x10FFFF` raise `UnicodeConversionError`; a non-codepoint element (`:oops`) raises `ArgumentError`. They perform faithful UTF-8 encoding only — no normalization/case-folding/identifier logic — so there is no confusable/NFC weakening to attack here (that lives in the tokenizer, not this file).
- **Body-recursive helpers** (`delete/2` at `list.ex:163`, `do_insert_at`/`do_replace_at`/`do_update_at`/`do_pop_at`). Non-tail recursion consumes heap-backed stack on the BEAM (no C-style overflow) bounded by list length — O(n) work/space on O(n) input, not asymmetric.
- **Index/length arithmetic** (`insert_at`, `replace_at`, `update_at`, `pop_at`, `improper?`). Linear traversals, bignum-safe index math, guarded by `is_list`/`is_integer`; out-of-range indices are handled (capped/return original) per docs.

## 3. Risks / caveats
- **`myers_difference` severity is usage-dependent.** Rated Low because reaching it needs an application to diff *untrusted* lists. If your deployment exposes list-diffing on a request path, treat it as higher: this pass instrumented **4.3 GB resident term memory plus ~29 s CPU for 16k total elements**, a real single-request memory+CPU DoS. The O(D²) path-memory claim is no longer asserted — it is measured (see finding evidence).
- **Scope.** I judged only the documented public API of this file. Reachability of the ruled-out sinks depends on whether *callers* forward untrusted data into them; that is a property of those call sites, not of `list.ex`. This remains an inherent property of reviewing a general-purpose library module — there are no in-file call sites to inspect — not an unverified assumption about a specific neighbour module. Confusable/bidi/NFC obligations from the folder threat model do not apply here — this file does no identifier or normalization handling (confirmed: `to_string`/`to_charlist` do only UTF-8 encoding, no folding).

## 4. Did I verify this?
- **myers memory (O(D²)):** Verified by execution — spawned-process `Process.info(:memory)`/`:heap_size` after the call, four input sizes, super-quadratic growth confirmed (4.3 GB @ 16k elements). **Upgraded from `asserted` to `verified`.**
- **`to_integer` scaling:** Verified by execution — four digit-string sizes up to 800 KB, exponent ≈ 1.5, input-proportional, ~74 ms ceiling for 800 KB. **Upgraded from hedged "out of scope" assumption to `verified` ruled-out non-finding.**
- **atom / unicode-conversion sinks:** Verified by reading the wrappers and (first pass) exercising the error paths; unchanged.
- Neither hedge required reading a sibling `src/` module — both bottom out in Erlang BIFs (`:erlang.list_to_integer`, BEAM heap accounting), which the sandbox toolchain (Erlang/OTP 29, Elixir) let me exercise directly.

### Dropped in re-review
None. Every prior finding and ruled-out entry is preserved; the sole finding was strengthened (memory evidence) and one ruled-out entry had its hedge replaced with measured evidence.
