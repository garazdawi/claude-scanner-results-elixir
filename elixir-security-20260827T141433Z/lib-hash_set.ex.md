# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/hash_set.ex
<!-- sha256: d6d1ab4cdf516045c62fb24bb2274c6004f15576e2ba6a1b1943b9615f55ad1a -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:00:24Z_

I have everything I need. The direct data-structure API is correct and crash-free (600k differential ops vs. MapSet); the collision angle is weak and inherent; the one real defect is the protocol dispatch. Here is the report.

---

## Summary

`HashSet`'s core data-structure API is correct, but every `Enumerable`/`Collectable`/`Inspect` operation is dead-on-arrival because the protocol impls dispatch to the atom `:HashSet` instead of the module `:"Elixir.HashSet"`, raising `UndefinedFunctionError` on documented public-API use.

## Security

#### `CSCAN-hash_set-protocol-dispatch-wrong-atom`
- **Finding ID:** `CSCAN-hash_set-protocol-dispatch-wrong-atom`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `lib/elixir/lib/hash_set.ex:282,288,294,309,328` — `Enumerable.HashSet.reduce/3`, `.member?/2`, `.count/1`, `Collectable.HashSet.into/1`, `Inspect.HashSet.inspect/2`
- **Class:** Protocol and behaviour dispatch
- **Issue:** Each protocol impl does `module = String.to_unsafe_atom("HashSet")` and then calls `module.reduce/put/member?/size/to_list`. `String.to_unsafe_atom/1` is `:erlang.binary_to_atom(s, :utf8)`, so `"HashSet"` becomes the atom `:HashSet`. The actual module atom is `:"Elixir.HashSet"` (`String.to_unsafe_atom("HashSet") == HashSet` is `false`). The `Elixir.` prefix is missing, so every dispatch targets a non-existent module `:HashSet`. (The sibling `hash_dict.ex:268-319` has the identical mistake, confirming a systematic refactor regression rather than a one-off.)
- **Impact / threat:** Every documented protocol operation on a `HashSet` fails deterministically — no special input required. `Enum.*(set)`, `for x <- set`, `Enum.into(_, HashSet.new())`, and `for ... into: HashSet.new()` raise an uncaught `UndefinedFunctionError` (availability: the caller's process crashes). `inspect(set)` degrades to a `#Inspect.Error<…>` blob by default and *raises* `Inspect.Error` under `inspect(_, safe: false)`. The trust boundary is availability/reliability: a legacy `HashSet` value that reaches generic enumeration, collection, or an error/log-formatting path (where `inspect` runs automatically) turns a benign value into a crash, and the broken `Inspect` dumps the raw internal struct (`%{size: …, root: {…}, __struct__: HashSet}`) instead of the intended `#HashSet<…>`. No confidentiality or integrity impact; this is fundamentally a correctness regression with a deterministic DoS dimension, scoped to the deprecated module.
- **Evidence:** `verified`.
  - `String.to_unsafe_atom("HashSet")` → `:HashSet`; `… == HashSet` → `false`.
  - `Enum.to_list(s)` / `Enum.member?(s,1)` / `Enum.count(s)` / `Enum.into([9], HashSet.new())` each → `RAISED UndefinedFunctionError: function :HashSet.reduce/3 (…member?/2 …size/1 …put/2) is undefined (module :HashSet is not available)`.
  - `inspect(s)` → `"#Inspect.Error<… function :HashSet.to_list/1 is undefined (module :HashSet is not available) … %{size: 2, root: {[], [], [1], [], [2], [], [], []}, __struct__: HashSet} …>"`; `inspect(s, safe: false)` → `RAISED Inspect.Error`.
  - Contrast: the direct API (`new/put/member?/size/union/subset?/equal?/to_list`, which do **not** go through `to_unsafe_atom`) all work, and `String.to_unsafe_atom("Elixir.HashSet") == HashSet` → `true` with `m.size/m.to_list` dispatching correctly.
- **Suggested fix:** Use the fully-qualified module name so the atom resolves to the module: `String.to_unsafe_atom("Elixir.HashSet")` at all five sites (and likewise `"Elixir.HashDict"` in `hash_dict.ex`). Since the module is already loaded at dispatch time, `String.to_existing_atom("Elixir.HashSet")` is equivalent and avoids even the appearance of unsafe atom creation.

### Ruled out
- **Hash-collision algorithmic-complexity DoS (`put`/`member?`)** — `key_hash/1` is `:erlang.phash2/1` (bounded to ~27 bits; verified max `134217504`), and colliding elements degrade a slot into a nested-tuple chain. *Not practically exploitable beyond what the recommended `MapSet` already exposes*: tree depth is capped at ~9 levels by the hash width unless attackers supply **full** 27-bit collisions, and those are expensive — scanning 8,000,000 integers (~7 s) yielded a largest full-collision bucket of only **5**. Cheaply-findable inputs keep lookups bounded (verified: 10k `member?` on the colliding set 174 µs vs 128 µs on a distinct set). The property is inherent to any `phash2`-based collection, not specific to this file. Basis: `verified`.
- **Atom-table exhaustion via `String.to_unsafe_atom`** — the argument is the compile-time constant `"HashSet"`; the resulting atom is fixed and pre-existing, so no unbounded atom growth from external input. (The call is nonetheless *wrong* — see the finding above.) Basis: `verified` (constant argument; single atom).
- **`do_delete`/`do_compact_node` crash via `delete/2`** — the compaction clauses are only reached with non-empty subnodes maintained by the insert/delete invariants; a 3000×200-op randomized differential fuzz against `MapSet` (600k ops) showed identical size/contents/membership and no `FunctionClauseError`/crash. Reaching the empty-template compaction path requires a forged internal `root`, which violates the `@opaque t` contract (excluded). Basis: `verified`.
- **Struct forgery (`%HashSet{root: …}`) reaching `elem/2`** — requires the caller to construct an invalid opaque struct, i.e. violate the documented type/`@opaque` contract; excluded by scope. Basis: `asserted` (contract-violation exclusion).
- **Compile-time execution** — the `Enum.each(0..(@node_size-1), …)` clause generation and the `message` binding operate only on module constants (`@node_size = 8`); no external input is read at build time. Basis: `verified` (constants only).
- **No code-eval / OS-command / file-path / network / archive / crypto / `binary_to_term` sinks** exist anywhere in the file. Basis: `verified` (full-file read).

## Risks / caveats
- Severity reflects a security review: the defect is a deterministic *availability* crash on documented public protocol operations, not a confidentiality/integrity issue, and it is not attacker-parameterized (it fires for any consumer). A maintainer weighting "deprecated module + no hostile-input requirement" heavily could reasonably triage it Low; I rate it Medium because it is guaranteed, reachable through the module's advertised protocol contract, and specifically breaks `inspect` in the error/log paths where robustness matters most.
- I could not consult git history (the working copy is not a git repo), so I confirmed the regression's nature by cross-checking the identical pattern in the sibling `hash_dict.ex` rather than by diffing against a prior revision.
- The collision analysis bounds practical exploitability empirically (largest full-collision bucket = 5 in 8M scanned) but does not prove an infeasibility theorem; a determined attacker with large offline `phash2` preimage search could build longer chains, with the same caveat applying equally to `MapSet`/Erlang maps.

