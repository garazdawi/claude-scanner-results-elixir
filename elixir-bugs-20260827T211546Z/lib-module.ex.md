# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module.ex
<!-- sha256: 9f30a3ab43608e96e71383a5eeb01a4df3d4212f5c5c2aeaa8c88aeab3f7b486 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module.ex
<!-- sha256: 9f30a3ab43608e96e71383a5eeb01a4df3d4212f5c5c2aeaa8c88aeab3f7b486 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (resolving first-pass "outside this file" / "out of scope" hedges by executing the runtime axis rather than reading neighbours)_

## Summary
`module.ex` is the Elixir standard-library `Module` API — a thin public surface over the `:elixir_module` / `:elixir_def` / `:elixir_overridable` / `Kernel.Typespec` compiler internals plus a local module-attribute ETS get/put/delete family. The prior report carried one dismissal whose `verified` basis rested **only on reading neighbour modules** (`elixir_module.erl`, `elixir_overridable.erl`) — the compiler-forwarding coverage caveat, which still listed `Not examined: runtime behaviour internal to the callee modules`. Per the re-review rule that *reading a neighbour never clears an axis*, that basis was not admissible as `verified`. In this pass I **executed** the runtime axis instead: I ran `create/3`, `overridables_in/1`, `defines_type?/2`, and `spec_to_callback/2` end-to-end under a live compiler and compared their outputs to the documented contracts. They match, so the entry is now `verified` on **executed** evidence with `Not examined: none`. The installed 1.21.0-dev source is byte-identical to the reviewed file (sha256 confirmed equal), so execution against it is valid. I still find no genuine, reachable bug.

## Bugfixes

No findings.

### Ruled out

- **`get_last_attribute/3` returning the wrong element for accumulating attributes** — `lookup_accumulate_attribute/4` uses `:ets.select(bag, ..., 1)`; suspicion was it returns the *first-inserted* (oldest) value. Ran it: `put 1; put 2` yields `get_last_attribute #=> 2`, consistent with `get_attribute #=> [2, 1]`. Confidence: N/A (dismissal). Basis: `verified` (ran `/tmp/attrs.exs`: `get_last acc: 2`, `get_attribute acc: [2, 1]`). Axes examined: insertion order, accumulate vs. plain | Not examined: none.

- **`delete_attribute/2` order/return-shape mismatch for accumulate vs. plain vs. unset** — all three documented outcomes checked live: accumulate returns reversed list `[2, 1]` and the key keeps accumulating afterward (`[]`); plain set value returns the value (`7`); never-set key returns `nil`. Confidence: N/A (dismissal). Basis: `verified` (ran `/tmp/attrs.exs`: `delete acc: [2, 1]`, `after delete acc: []`, `delete plain: 7`, `delete missing: nil`). Axes examined: accumulate/plain/unset, post-delete state | Not examined: none.

- **`get_attribute/3` default handling** — the `[] when is_integer(caller_line)` warning clause is unreachable from public `get_attribute/3`, which passes `caller_line = nil`; the plain `[] -> default` clause is taken. Confidence: N/A (dismissal). Basis: `verified` (ran `/tmp/attrs.exs`: `get default: :dflt`). Axes examined: caller_line nil vs integer | Not examined: none.

- **Executable documentation mismatches** — all executable `@doc` examples exercised. The attribute doctests (`get_last_attribute`, `get_attribute`, `delete_attribute`, `overridables_in`) reproduce exactly under a live compile. Confidence: N/A (dismissal). Basis: `verified` (ran `/tmp/attrs.exs` and `/tmp/fwd.exs`; `overridables_in: [bar: 0, foo: 0]` matches the doctest at lines 1358–1366). Axes examined: executable `@doc` examples on the attribute/overridable paths | Not examined: none.

- **Signature-building / `merge_signature` / `simplify_arg` arithmetic (lines 1064–1229)** — reachability re-confirmed by in-file grep: `build_signature/2` (line 1882) and `merge_signatures/3` (line 1907) are called *only* from `compile_definition_attributes/6` (line 1854, `@doc false`) and, via the doc-signature path, from `add_doc/6` (line 1841, `@doc false` **and** `@deprecated "Use @doc instead"`). No `@doc`-bearing, non-deprecated public function reaches this code. This is a within-file callgraph fact, confirmed by reading this file (not assumed, not from a neighbour). Confidence: N/A (dismissal). Basis: `verified` (grep of `build_signature`/`merge_signatures` callers at lines 1882/1905–1907; both callers carry `@doc false` at lines 1839 and 1851, `add_doc` additionally `@deprecated` at line 1840 — read at lines 1839–1865). Axes examined: caller set, doc-visibility of callers | Not examined: none.

- **`split/1` raising on `Module.split(Elixir)` / Erlang-style atoms** — raising `ArgumentError` on a non-`Elixir.`-prefixed atom is the documented contract, enforced by the `split(_module, original)` fall-through clause (line 1835–1836). Confidence: N/A (dismissal). Basis: `asserted` (matches `@doc` and the fall-through clause; documentary contract — the only input axis, prefixed vs. non-prefixed atom, was **not pushed through a live comparison** this pass, so it stays asserted rather than being over-claimed as verified). Axes examined: none executed | Not examined: `Elixir.`-prefixed vs. non-prefixed atom at runtime.

- **Compiler-forwarding entry points `create/3`, `spec_to_callback/2`, `defines_type?/2`, `overridables_in/1`** — *(Basis re-grounded in re-review: the prior pass marked this `verified` on the strength of reading `elixir_module.erl`/`elixir_overridable.erl`, which does not clear the runtime axis. I replaced that with executed evidence.)* Each forwarder was run end-to-end against the live compiler:
  - `create/3` (line 939/943/952) → returns the documented `{:module, module, binary, term}`; the resulting module is loadable and its function runs. Both the `%Macro.Env{}` and keyword-`opts` clauses exercised.
  - `overridables_in/1` (line 1371) → returns `[bar: 0, foo: 0]`, matching `@spec :: [definition]` and the module doctest.
  - `defines_type?/2` (line 1301) → returns a boolean (`false` for an undefined type), matching `@spec :: boolean`.
  - `spec_to_callback/2` (line 1316) → returns `true` for an existing spec and `false` for a missing one, matching `@spec :: boolean`.

  The `missing :file` guard on `create/3` (line 944–946) and the `arity` range guards (`0 ≤ arity ≤ 255`) on the typespec forwarders are applied before forwarding. Confidence: N/A (dismissal). Basis: `verified` (ran `/tmp/fwd.exs` → `create/3: {MyDynMod, 1368, :world}`, `create/3 env: {MyDynMod2, 1}`, `overridables_in: [bar: 0, foo: 0]`, `defines_type?: false`; ran `/tmp/spec.exs` → `spec_to_callback existing/missing: {true, false}`). Axes examined: runtime return shape/value for all four forwarders, both `create/3` argument clauses | Not examined: none.

## Risks / caveats
The reviewed file is byte-identical to the installed, compiled 1.21.0-dev source (sha256 `9f30a3ab…` matches `/opt/elixir/lib/elixir/lib/module.ex`), so the live executions above are executions of the exact bytes under review — mature upstream code, low prior probability of a reachable public-API bug, consistent with the empty Bugfixes result. The one previously read-only "outside this file" hedge (compiler-forwarding coverage) is now closed by executing the forwarders themselves rather than by reasoning about neighbour internals; behaviour strictly *internal* to `:elixir_module` / `:elixir_def` / `:elixir_overridable` / `Kernel.Typespec` remains the province of those files' reviews, but `module.ex`'s own obligation — accepting documented inputs and producing documented outputs at the boundary — was exercised directly and holds.

## Did I verify this
- Confirmed the reviewed path resolves to byte-identical installed source via `sha256sum` (equal to the recorded hash), and read the forwarding/attribute regions there (lines 930–956, 1295–1374, 1836–1865).
- Ran the four compiler-forwarding entry points live (`/tmp/fwd.exs`, `/tmp/spec.exs`) — this replaces the prior read-only basis and clears the runtime-behaviour axis with executed output.
- Re-ran the attribute get/put/delete/default and overridable doctest paths live (`/tmp/attrs.exs`); every output matches the `@doc` examples.
- Grepped in-file for every caller of `build_signature`/`merge_signatures` and read lines 1839–1865 to confirm both live only under `@doc false` (and, for `add_doc`, `@deprecated`) functions.

### Dropped in re-review
None. Every first-pass entry is preserved. One change of *basis quality*: the compiler-forwarding dismissal was previously `verified` on the disallowed ground of reading neighbour modules; it remains a dismissal but is now `verified` on **executed** evidence (the forwarders were run), with `Not examined: none`. The `split/1` entry stays `asserted` (its input axis was not pushed through a live comparison this pass), which is a deliberate non-upgrade consistent with "prefer downgrading."

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Executable documentation mismatches** — ran `doctest Module` under ExUnit; all 11 doctests pass (`reserved_attributes/0`, `concat/1,2`, `safe_concat/1,2`, `split/1`, `behaviour_info`/`module_info` moduledoc examples). Confidence: N/A (dismissal). Basis: `verified` (ran `/tmp/dt.exs`, `Result: 11 passed`). Axes examined: all executable `@doc` examples | Not examined: none.
