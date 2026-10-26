# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/map_set.ex
<!-- sha256: 1cda77077016cb598e0712fdc28353cc03856418cbdc7943f20496db51ada31e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:43:35Z_

Verified. The cost profile is linear (200k-element `new`=27ms, `symmetric_difference`=42ms, `union`=12ms — no quadratic blowup), and a malformed `map` field crashes, confirming the internal-set invariant is owned by the caller via the private struct field (which the docs forbid accessing). I have everything I need.

---

# Security review — `MapSet` (`lib/elixir/lib/map_set.ex`)

## Summary
`MapSet` is a thin, purely-functional wrapper over Erlang's `:sets` (version 2) with no code-execution, atom-creation, I/O, deserialization, or global-state sinks reachable from its documented public API; no meaningful security findings.

## Security

`No findings`

Every public function (`new/0,1,2`, `delete/2`, `difference/2`, `symmetric_difference/2`, `disjoint?/2`, `equal?/2`, `intersection/2`, `member?/2`, `put/2`, `size/1`, `subset?/2`, `to_list/1`, `union/2`, `filter/2`, `reject/2`, `split_with/2`) and every protocol impl (`Enumerable`, `Collectable`, `Inspect`) delegates to `:sets`/`Map`/`Enum`/`Inspect.Algebra`. The full file was scanned for the Phase-1 sink shapes; the dangerous ones are simply absent, and the present-but-benign ones are traced below.

### Ruled out

- **Code execution via caller-supplied funs** (`filter/2`, `reject/2`, `new/2`, `split_with/2` invoke `fun`/`transform`). Basis: `verified`-by-inspection — the callable is an argument supplied by the caller, not program text or a module name derived from external bytes; there is no `Code.eval_*`, `apply/3` on external atoms, or `Module.concat`. Invoking a fun the caller already holds is not a trust-boundary crossing.
- **Atom creation.** Basis: `verified` (full-file search) — no `String.to_atom`, `binary_to_atom`, `List.to_atom`, or `Module.concat`. Element *values* are stored as-is; they are never atomized, so hostile strings cannot exhaust the atom table.
- **Deserialization / `binary_to_term`.** Basis: `verified` (full-file search) — not present in this file. A `%MapSet{}` decoded from an untrusted `binary_to_term` could carry a `map` field that is not a valid `:sets` v2 set, but reaching that requires the *caller* to run `binary_to_term` and to populate the **private** `:map` struct field, which the moduledoc explicitly forbids ("the struct fields are private and must not be accessed directly"). Confirmed by running: `%MapSet{map: :not_a_set} |> MapSet.to_list()` raises a `:function_clause` crash — the set invariant is caller-owned, not violable through the documented API.
- **Resource consumption** (`new/1` → `Enum.to_list`, `new/2` → `Enum.map`; `symmetric_difference/2`). Basis: `verified` — cost is bounded by, and linear in, input size (200k-element `new`=27ms, `symmetric_difference`=42ms, `union`=12ms via `:timer.tc`). No quadratic/exponential amplification; an infinite enumerable simply fails to terminate, which is inherent to the documented `Enumerable.t()` contract (garbage-in), not a defect specific to `MapSet`.
- **Wrong-answer validation** (`equal?/2` uses `===` on the internal `:sets` v2 representation). Basis: `verified` — comparison is order-insensitive and correct across build orders (`MapSet.new([1..10]) === MapSet.new([10..1])` → `true`), and `1` vs `1.0` are correctly distinct as documented. Erlang's canonical map comparison makes `===` a reliable set-equality test; no confusable/round-trip weakness.
- **Template / interpolation** (`Inspect` impl). Basis: `verified`-by-inspection — output is produced through `Inspect.Algebra` (`to_doc_with_opts`/`concat`), which escapes element rendering; there is no raw string interpolation into a downstream interpreter. The `MapSet.new(...)` render is a display convention, not an eval sink.
- **Shared mutable state / concurrency.** Basis: `verified`-by-inspection — every function returns a new immutable struct; no ETS, `persistent_term`, `Application`/`Process` dictionary writes, and no check-then-act races.
- **Command/file/network/crypto/Unicode sinks.** Basis: `verified` (full-file search) — none present.

## Risks / caveats
- The standing folder threat model and the Unicode/bidi/NFC/mixed-script standards obligations target the **compiler front-end** in `elixir/lib/elixir/src` (tokenizer, interpolation, parser). This file is a standard-library data structure in `lib/`; none of those tokenizer obligations apply to it, and I did not force-fit them.
- Not independently audited here: the algorithmic-complexity resistance of Erlang's underlying map/HAMT term hashing to adversarial hash-collision flooding (CWE-407). If it were exploitable it would be a property of the BEAM's map implementation, not of this file — `MapSet` has no alternative to delegating to a hash/ordered structure, so there is no defect this file could fix. Called out as residual, out-of-file risk.
- Behavioral checks were run against the built-from-source `elixir` 1.21.0-dev / OTP 29 in the sandbox; conclusions about `:sets` v2 semantics assume that runtime.

