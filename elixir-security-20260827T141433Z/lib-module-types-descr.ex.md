# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/descr.ex
<!-- sha256: 1057797db27539ddf6ae0f4ffcf27371643a0ac8322318e3c4cee97b9bf306ad -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/descr.ex
<!-- sha256: 1057797db27539ddf6ae0f4ffcf27371643a0ac8322318e3c4cee97b9bf306ad -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (hedges resolved with executed evidence + neighbour reads)_

# Security review — `Module.Types.Descr` (`descr.ex`)

## 1. Summary
The set-theoretic type engine is internally sound, but the tuple-index operations materialize a `term()` list whose length equals an unbounded, source-controlled integer, giving a compile-time memory-exhaustion DoS (finding #1). A second, independently-triggered compile-time DoS is now confirmed: constructing a **deeply nested tuple type** is **quadratic** in nesting depth because `bdd_leaf_new/2` re-hashes the full element subtree at every level, and the parser imposes **no** nesting limit (finding #3 — upgraded from a hedged assumption in the first pass). A bounded atom-creation amplification in the type-rendering path also exists (finding #2).

All three hedges flagged from the first pass are resolved below: the apply.ex reachability hedge is verified by reading `apply.ex`; the "gated by parser nesting limits" assumption is falsified by execution and the recursion hedge is upgraded to finding #3; the set-operation-cost caveat is retained but now subsumed for the nested-literal case by finding #3.

## 2. Security

#### `CSCAN-descr-tuple-index-alloc-dos`
- **Finding ID:** `CSCAN-descr-tuple-index-alloc-dos`
- **Severity:** High
- **Confidence:** high
- **Location:** `descr.ex:5642` (`tuple_of_size_at_least_static?/2`), `descr.ex:5634` (`tuple_of_size_at_least/1`), `descr.ex:5127` (`tuple_fill/2`); reached via public `tuple_delete_at/2` (5408), `tuple_insert_at/3` (5497), `tuple_replace_at/3` (5624).
- **Class:** Resource consumption (CWE-770/400).
- **Issue:** `tuple_of_size_at_least_static?(descr, index)` computes `List.duplicate(term(), index)` (line 5642) and `tuple_of_size_at_least(index+1)` computes `List.duplicate(term(), n)` (line 5634) whenever the cheap size proof `tuple_bdd_positive_size_at_least?/2` fails. The public entry points guard only `is_integer(index) and index >= 0` (lines 5408, 5497, 5624) — there is no upper bound on `index`. Allocation grows linearly and without limit with the index value.
- **Impact / threat:** The type checker runs during `mix compile` (including on dependencies) and in language servers. **Reachability now verified by reading `apply.ex`** (see "Did I verify this"): `apply.ex` dispatches every tuple builtin (`:erlang.delete_element/2` → 546-554, `:erlang.insert_element/3` → 496-506, `:erlang.setelement/3` → 521-531, and the `Tuple.*`/`Kernel.*` wrappers at 470, 569, 590, 621, 646) with a raw `when is_integer(index)` literal taken straight from the source AST, and — critically — **each dispatch itself pre-allocates `open_tuple(List.duplicate(term(), max(index-1,0)) ++ ...)`** (e.g. `apply.ex:471,498,523,548,570,591,622,647`) *before* calling into descr. So the root cause is shared and amplified across the boundary. A single line such as `Tuple.delete_at(t, 50_000_000)` makes the compiler allocate/spin without bound. Trust boundary crossed: compiling/type-checking untrusted Elixir source (a dependency, or a file opened in an LSP).
- **Evidence:** `verified` (re-run end-to-end through the real compiler this pass).
  - `Code.compile_string("... Tuple.delete_at(t, N) ...")`: `N=3` → **22.7 ms**; `N=2_000_000` → **779.3 ms**; `N=8_000_000` → **2658.4 ms**. Clean linear scaling in the source index, through the public `mix compile` path.
  - Prior-pass isolated descr-only measurements corroborate (`peak_delta_MB` linear in index; `Tuple.delete_at(t, 20_000_000)` → `deltaMB≈3012`).
- **Suggested fix:** Answer the size question without building an `index`-length list. In `tuple_of_size_at_least_static?/2`, when the positive proof fails, compare `index` against the largest tuple-literal arity actually present in the BDD leaves and short-circuit to `false`. Likewise guard `tuple_of_size_at_least/1` and `tuple_fill/2`. The identical `List.duplicate(term(), index)` pre-allocation in the `apply.ex` callers must be fixed in the same change (out of scope for this file but same root cause).

#### `CSCAN-descr-nested-tuple-quadratic-dos`  *(Upgraded from hedged assumption in first pass)*
- **Finding ID:** `CSCAN-descr-nested-tuple-quadratic-dos`
- **Severity:** Medium
- **Confidence:** medium
- **Location:** `descr.ex:5691-5692` (`bdd_leaf_new/2`), reached from `tuple_new/2` (5781) → `tuple_descr_static/2` (4763) → `tuple_descr/2` (4750); driven from source by nested tuple *value* expressions inferred via `of.ex`/`expr.ex`.
- **Class:** Resource consumption (CWE-407 — algorithmic complexity / quadratic blowup).
- **Issue:** `bdd_leaf_new(:closed, arg2)` computes `-:erlang.phash2(arg2)` and `bdd_leaf_new(tag, arg2)` computes `:erlang.phash2([arg1 | arg2])` over the **entire** element list (5691-5692). When the type checker infers the type of a nested tuple literal `{{{…x…}}}` of nesting depth `d`, it builds a tuple leaf at each of the `d` levels, and each leaf hashes its full nested-subtree content (size O(d−k) at level k). Total work is Σ O(d−k) = **O(d²)**. There is no memoized structural sharing at the hash step to avoid re-hashing the already-hashed inner leaf.
- **Impact / threat:** Compile-time DoS on untrusted source. The first pass assumed this path was "gated upstream by the parser's own nesting limits" — **that assumption is false** (verified by execution: `Code.string_to_quoted` accepts nesting depth 500 000 in 321 ms, linearly, with no nesting cap). So a single source line binding a deeply nested tuple literal drives the quadratic directly. Same trust boundary as finding #1 (compiling untrusted Elixir).
- **Evidence:** `verified` (executed this pass).
  - Isolation test, `Code.compile_string("def f(x), do: <nested>")`:
    - Nested **tuples** `{{{…x…}}}`: depth 500 → **43.3 ms**, 1000 → **110.0 ms**, 2000 → **429.0 ms** (≈4× time for 2× depth ⇒ quadratic); depth 100 000 → **timed out > 2 min**.
    - Nested **parens** `(((…x…)))` at the *same* depths (identical parse-nesting, but no tuple descr built): flat **~1.5 ms** at all depths.
    - The differential isolates the super-linear cost to the tuple-descr construction path, not parsing/expansion/codegen.
  - Parser has no gate: `string_to_quoted` of depth 500 000 → 321.8 ms (linear).
  - Code mechanism confirmed at `descr.ex:5691-5692` (phash2 over full element content per leaf).
- **Why medium/medium (not high):** the blowup is quadratic (not exponential) and requires the attacker to ship a physically large, deeply nested literal, so the constant factor per byte of source is modest compared with finding #1's one-integer trigger; and I confirmed the mechanism for **tuples** specifically. Maps and non-empty lists build leaves through the same `bdd_leaf_new` and are very likely to share the defect, but I did not separately execute the map/list variants — hence not high confidence on scope.
- **Suggested fix:** Make leaf identity cheap and compositional: derive a leaf's hash from its children's already-computed hashes (e.g. `phash2({tag, [child_hash…]})`) instead of re-hashing the full recursive term, so building a depth-`d` structure is O(d). Alternatively cap/short-circuit descr construction depth with a diagnostic.

#### `CSCAN-descr-alias-segment-atom-creation`
- **Finding ID:** `CSCAN-descr-alias-segment-atom-creation`
- **Severity:** Low
- **Confidence:** high
- **Location:** `descr.ex:1255`, `literal_to_quoted/1`; reachable via `atom_to_quoted/1` (1264) → `to_quoted/2` (739) / `to_quoted_string/2`.
- **Class:** Atom creation (CWE-770).
- **Issue:** When rendering an atom type whose value is alias-classified (`Macro.classify_atom(lit) == :alias`, line 1246), the code splits on `.` and interns every segment with `String.to_unsafe_atom/1` (line 1255, no existence check). An alias atom created *as a whole* (e.g. source literal `:"Elixir.A.B.C…"` or one built via `String.to_atom/1`) has segments that were never interned, so rendering creates new atoms.
- **Impact / threat:** Type-mismatch diagnostics render types via `to_quoted_string/2`, so compiling untrusted source that triggers a warning involving such an atom forces segment interning — an amplification of the never-GC'd atom table.
- **Evidence:** `verified` (prior pass). One 40-segment alias → exactly 40 new atoms. Amplification is bounded: atoms cap at 255 bytes (`String.to_atom` of a 50 000-segment alias raised `SystemLimitError`), so ≤ ~124 segments (≤ ~124 new atoms) per source alias atom.
- **Suggested fix:** Prefer `String.to_existing_atom/1` where possible, or document why `to_unsafe_atom` is acceptable given the ≤255-byte bound and diagnostic-only gating.

### Ruled out
- **`fun/1` large-arity rendering** (`fun_bdd_to_pos_dnf/2` `List.duplicate(none(), arity)` at 2097; `domain_to_flat_args/2` at 323): same "list sized by an integer" shape and it *does* blow up in isolation (`to_quoted_string(fun(5_000_000))` → 9.4 s), but function arities are BEAM-capped at 255 and captures `&f/N` above that are rejected, so no source path supplies a large arity. Basis: `verified` (BEAM arity limit) — not reachable, defense-in-depth only. *(Not re-opened this pass; I did not attempt a typespec-with-256-args path — see caveat.)*
- **`maybe_struct/1` → `struct.__info__(:struct)`** (line ~4587): dispatches on a `:__struct__` atom taken from the type during rendering, but the atom must already exist, the call is limited to the side-effect-free auto-generated `__info__(:struct)`, and it is wrapped in `try/rescue`. No arbitrary code execution. Basis: code reading.
- **BDD `:erlang.phash2` node identity** (`bdd_leaf_new/2` at 5691, `bdd_compare/2` at 5704): a hash collision cannot cause a false `:eq` — `bdd_compare` compares the full leaf tuple `{hash, tag, content}`, so distinct contents with equal hash still order strictly. No correctness/security impact. Basis: code reading. *(Note: this same full-content hashing is the performance defect in finding #3 — correct, but quadratic.)*
- **`fun_from_inferred_clauses([])` / `Enum.reduce/2` on empty** (line ~233): raises on an empty clause list, but that is a degenerate internal input (a function always has ≥1 clause) with no attacker-meaningful effect. Basis: code reading.

## 3. Risks / caveats
- **Reachability of finding #1 is verified**, not assumed: `apply.ex` maps `Tuple.delete_at/2`, `Tuple.insert_at/3`, `Kernel.put_elem/3`, `Kernel.elem/2`, `:erlang.setelement/3`, `:erlang.insert_element/3`, `:erlang.delete_element/2`, `:erlang.element/2` onto these functions with literal integer indices from the AST, and pre-allocates the same `List.duplicate(term(), index)` before dispatch (`apply.ex:470-653`).
- **Set-operation cost (retained caveat):** `phi`/`bdd_*`/emptiness routines are worst-case exponential in the *size* of the types (the code comments acknowledge "can grow exponentially"). This is inherent to set-theoretic subtyping and bounded by how large a type the source literally writes. For the specific *nested-literal* case this is now concretized as finding #3 (quadratic). I did not construct a source input that triggers the *exponential* union blowup, so that residual remains an unquantified caveat rather than a finding.
- **Map/list nested-literal variants:** finding #3 is executed and confirmed for tuples. Maps and non-empty lists construct leaves through the same `bdd_leaf_new/2` and are very likely to exhibit the same O(depth²) behavior, but I did not separately execute those variants this pass.
- **Scope:** I re-read the descr sinks and the `apply.ex` caller graph in full this pass. I did not audit `expr.ex`, `of.ex`, or `pattern.ex` beyond confirming they feed nested tuple/value types into `tuple_descr`/`bdd_leaf_new` (finding #3's driver) and confirming `apply.ex`'s builtin dispatch (finding #1's driver).

## 4. Did I verify this
- **apply.ex reachability hedge → verified.** Read `apply.ex:236-660`, 737-943. Every tuple builtin (`delete_element`, `insert_element`, `setelement`, `element`, and the `Tuple.*`/`Kernel.*` wrappers) dispatches with a raw source-literal `index` under `when is_integer(index)` and pre-allocates `List.duplicate(term(), …)`. Hedge resolved: reachability confirmed by reading, not assumed.
- **"gated by parser nesting limits" assumption → falsified by execution.** Ran `Code.string_to_quoted` on nesting depth 500 000 → 321.8 ms, linear, no rejection. The parser imposes no nesting cap.
- **"did not exhaustively prove no unguarded deep-recursion path" hedge → resolved by execution + code, upgraded to finding #3.** Isolation test (nested tuples quadratic 43/110/429 ms at depth 500/1000/2000; identical-depth parens flat ~1.5 ms; depth 100 000 tuples time out > 2 min) plus the `bdd_leaf_new/2` phash2-over-full-content mechanism at `descr.ex:5691-5692` establish an O(d²) source-controlled compile-time DoS. No longer an assumption.
- **Finding #1 magnitude → re-verified this pass** end-to-end through `Code.compile_string` (22.7 / 779.3 / 2658.4 ms at index 3 / 2M / 8M).
- **Not verified this pass (honestly flagged):** the map/list variants of finding #3 (mechanism shared, not executed); the exponential set-union blowup caveat (no triggering source constructed); a hypothetical typespec-with-huge-arity path around the `fun/1` ruled-out entry (not attempted — the entry remains ruled out on the BEAM 255-arity basis).

### Dropped in re-review
None. All three prior findings/entries are retained. The prior "structural recursion depth" risk caveat has been **promoted** (not dropped) into finding #3 with executed evidence.
