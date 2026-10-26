# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/apply.ex
<!-- sha256: 8b9b72bb778ed51944c188e5d05bc27ec73251057140782effc4d8cd4ecb22b3 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:57:12Z_

I now have everything I need, verified by running against the byte-identical built compiler. Here is the report.

---

# Security review — `lib/elixir/lib/module/types/apply.ex`

## 1. Summary
The compile-time type checker builds tuple *type* nodes whose size is taken directly from an integer literal in the source, with no upper bound — so a single innocuous-looking line of source (e.g. `put_elem(x, 40_000_000, :v)`) drives the compiler to multi-GB memory use and kills the VM, a verified compile-time denial of service.

## 2. Security

#### `CSCAN-apply-unbounded-tuple-type-alloc`
- **Finding ID:** `CSCAN-apply-unbounded-tuple-type-alloc`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `apply.ex:570` (`do_remote(Kernel, :elem, ...)`) and the sibling `do_remote/8` clauses at lines `471, 493, 498, 523, 548, 591, 617, 622, 647`.
- **Class:** Resource consumption (unbounded allocation driven by external input).
- **Issue:** Every tuple index/size builtin resolves the tuple’s *type* by materialising `List.duplicate(term(), N)` (then `open_tuple/1` / `tuple/1`) where `N` is the caller-supplied integer **literal** from the call site:
  - `element` (471), `insert_element` (498), `setelement` (523), `delete_element` (548), `elem` (570), `put_elem` (591), `insert_at` (622), `delete_at` (647) — all `List.duplicate(term(), max(index±k, …))`;
  - `make_tuple` (493), `Tuple.duplicate` (617) — `List.duplicate(elem_type, size)`.

  There is **no upper bound** on `index`/`size`, and the allocation happens *before* the `if index < 1 / < 0` validation. This directly contradicts the neighbouring, deliberately-bounded path: `is_data_size/2` (line 738) caps `tuple_size` comparisons to `literal in 0..15`, with the comment at lines 734–735 *“Limit the size of tuples to 16 entries as otherwise we may create large nodes.”* The tuple element/index/size operations were missed by that guard.
- **Impact / threat:** A single line such as `def f(x), do: elem(x, 40_000_000)` makes the *compiler* (not the running program) allocate a tuple type with tens of millions of `term()` entries. Runtime cost of the identical expression is zero (it just raises), so this is a pure compile-time cost asymmetry. Trust boundary crossed: **source text → compiler**, reached through documented compilation entry points (`mix compile`, `Code.compile_string/2`, `Code.string_to_quoted/2` + compile). Any pipeline that compiles/type-checks less-trusted Elixir is affected: a dependency or generated `.ex` file containing such a literal hangs/OOMs `mix compile`, and a language server that type-checks a file on open/save (type-checking runs *after* macro expansion, so no attacker macro code need execute for this trigger) can be OOM-killed merely by opening a hostile source file. The standing threat model lists “DoS of the compiling process (crash, unbounded CPU/memory)” as in scope.
- **Evidence:** `verified` (built compiler is byte-identical to the file under review — `diff -q` → `IDENTICAL`).
  - Linear scaling with the literal, via `Code.compile_string`:
    ```
    index=100      compile_ms=32.0
    index=1000000  compile_ms=199.2
    index=3000000  compile_ms=619.3
    index=8000000  compile_s=4.4
    index=15000000 compile_s=9.07     # ≈0.6 µs/element
    ```
  - Runtime of the same op is free: `:erlang.element(3_000_000, {:a,:b})` → `runtime elem(3_000_000) micros=0`.
  - Affects the whole family (all O(1) at runtime): `put_elem`/`setelement`/`insert_at`/`delete_at` at index 4M → `compile_ms` 2361 / 1663 / 1237 / 1226.
  - Memory / crash: peak `beam` RSS during a `put_elem(x, 20_000_000, :v)` compile = **9946 MB** (box has 11.9 GB). At index 40M the compile did not finish in 120 s in one trial and in another the **VM was aborted with no output even inside `try/rescue/catch`** (exit ≠ 0), i.e. an uncatchable OOM/`system_limit` kill of the compiler process.
- **Suggested fix:** Bound the constructed tuple size the same way `tuple_size` already is. Before calling `List.duplicate`, clamp/short-circuit: if `index`/`size` exceeds a small cap (reuse the “large nodes” threshold), skip building the sized tuple and fall back to `open_tuple([])` (for index ops) / emit the normal `:badindex`/domain error, and move the `index < 1` / `index < 0` negative checks *above* the `List.duplicate` call so out-of-range indices never allocate.

### Ruled out
- **Atom exhaustion** — no `String.to_atom`/`binary_to_atom`/`Module.concat` on external bytes anywhere in the file; every atom in the `signature/*` tables is a compile-time literal, and atoms handled at runtime (`remote`, `signature/6`, `remote_capture`) are already-interned module/function atoms from the parsed AST. Basis: read the full file; grepped the sink shapes.
- **Code execution / module load side-effects** — `Code.ensure_loaded?/ensure_loaded` (1618, 2234, 2390) and the `mod.module_info(:attributes)` / `mod.__protocol__(:impls)` / `module.__info__(:struct)` reflection (1619, 2235–2276) only run inside `format_diagnostic`/`export` on a module *named in the source being compiled*, which is already a compiled, code-path-resident (trusted) module, and only invoke compiler-autogenerated reflection functions. No `Code.eval*`/`compile_string`/`Module.create` on external input. Not an escalation over the arbitrary-macro-execution a compiled dependency already has.
- **Template/interpolation & ANSI** — `format_diagnostic` interpolations feed compiler *diagnostics* (stderr/logs), not a downstream interpreter; values are rendered through `to_quoted`/`Code.Formatter`/`Exception.format_mfa` (AST/type rendering, not raw bytes), and `ansi_red` is intentional and gated on `IO.ANSI.enabled?()`. No SQL/shell/markup sink.
- **Unbounded recursion / union blowup** — `filter_domain`, `apply_clauses`, `apply_infer`/`apply_strong`, `indexes_to_arrows`, `zip_*`, and the `remote_capture` reduce are linear in clause-count or arity; the union-producing paths cap at `@max_clauses` (→ `dynamic()`). No source-controlled depth beyond parser-bounded arity.
- **Type-system soundness of `subtype?`/`disjoint?`/`compatible?`** — a wrong answer weakens a *diagnostic* (false positive/negative warning); Elixir remains dynamically typed and enforces at runtime, so this is a correctness, not security, concern. Out of scope.
- **Shared mutable state / concurrency** — all mutation is on the immutable `context` via `put_in`/`update_in`; no ETS/`persistent_term`/process-dictionary writes and no check-then-act races in this file (parallelism lives in `ParallelChecker`).

## 3. Risks / caveats
- **Severity judgement:** rated Medium, not High, because the payload must be a large integer *literal* in the source (the `do_remote` guards require `is_integer(index/size)`, so the multiplier can’t come from a runtime value), the impact is DoS only (no code execution or disclosure), and the strongest “malicious dependency” vector is largely subsumed by the arbitrary compile-time macro execution a dependency already has. The language-server / “open a hostile file for review” vector is `asserted` (I reproduced the compiler DoS via `Code.compile_string`, which is the same type-checking path a language server drives, but did not drive a specific LS such as ElixirLS end-to-end).
- **Amplification factor:** peak RSS of ~9.9 GB at index 20M implies ~500 bytes of transient allocation per element (far above the raw cons-cell cost), i.e. the `open_tuple`/descr normalization amplifies memory well beyond the list itself; the exact threshold at which a given machine OOMs will vary, but the unbounded growth is the point.
- I reviewed only this file; predicates and constructors it relies on (`open_tuple`, `tuple`, `map_update`, `fun_apply`, `subtype?`, etc.) live in `Module.Types.Descr`/`Helpers` and were treated as black boxes — I did not audit them for their own resource bounds.

