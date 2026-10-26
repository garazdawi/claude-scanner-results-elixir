# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_for.erl
<!-- sha256: b5367cdcd8e7f3fc7bff244ea7384a3e8325fdd16a89e26731604aa28d9d2fae -->

_Analysed on 2026-08-27T16:48:47Z_

All ten comprehension code paths (enum/bin generators, guards, filters, `into: %{}`, `uniq`, `reduce`, `%MapSet{}`, bound/unbound bitstring size, `bc`, general `Collectable`) compile and evaluate correctly with no crash. The translation is a pure, deterministic AST→AST pass operating on input already normalized by `elixir_expand`. Here is the report.

---

# Security review: `elixir/lib/elixir/src/elixir_erl_for.erl`

## Summary
`elixir_erl_for.erl` is the compile-time lowering of Elixir `for` comprehensions into Erlang abstract format; it is a pure AST→AST transformation over input already validated and normalized by `elixir_expand`, contains no external-data sinks (no atom creation, deserialization, I/O, shared state, crypto, or NIF boundary), and I found no security defects.

## Security
No findings.

This file is reached only via `elixir_erl_pass:translate/3` (`elixir_erl_pass.erl:188-189`) during the Elixir→BEAM code-generation phase, i.e. *after* tokenization, parsing, and macro expansion. Its inputs are:
- `Meta` — source annotations,
- `Args` — the expanded `for` AST, and
- `S` — the `#elixir_erl{}` compiler scope.

Per the folder invariant, raw bytes are sanitized in `elixir_tokenizer`; by this pass the AST is well-formed Erlang-abstract-format-bound quoted terms. The module never atomizes, deserializes, evaluates external text, touches the filesystem/network, spawns OS processes, reads/writes process-, ETS-, or application-global state, or crosses a NIF boundary (confirmed by grep: no `binary_to_term`/`list_to_atom`/`binary_to_atom`/`apply`/`os:cmd`/`open_port`/`spawn`/`ets:`/`file:`/`put`/`get`/`erase`/`load_nif`/`rand_uniform` — zero matches). Every atom emitted into the generated code is a source-code literal (`badarg`, `cont`, `done`, `halt`, `true`, `__struct__`, `Elixir.MapSet`, …). All recursion (`translate_gen`, `build_reduce_each`, `collect_filters`, `join_filters`, `valid_var_size`, `no_unbound_var`) is bounded linearly by the number of generators/filters/segments in the source comprehension — no super-linear compile-time cost and no runtime-attacker-influenced compile loop.

## Risks / caveats
- **Correctness-as-security caveat.** The residual risk in a code-generator is *miscompilation* — emitting Erlang forms with different semantics than the source `for`. I exercised all major branches end-to-end (enum/bitstring generators, `when`-guards, filters, `into: %{}`, `into: MapSet.new()`, `uniq: true`, `reduce:`, bound and unbound bitstring size vars, `bc` binary comprehensions, general `Collectable`) and all produced correct results. This is behavioral spot-checking, not exhaustive proof of semantic equivalence; a subtle lowering bug would surface as a program-logic defect in downstream Elixir code, not as a memory-safety or injection primitive (the BEAM remains memory-safe), so its security impact would be indirect and application-specific.
- **Out-of-file dependencies (enumerated).** The "no crash on malformed `Args`" conclusion depends on `elixir_expand.erl:854-879` normalizing every `for` so the last arg is `[{do, Expr} | Opts]` and generators start with `<-`/`<<>>` (`assert_generator_start`, `validate_for_options`). If a caller invoked `elixir_erl_for:translate/3` directly with a malformed shape, `split_last`'s `{Cases, [{do, Expr} | Opts]}` match or the `translate_gen` clauses could `badmatch`/`function_clause` — but that requires calling a compiler-internal, non-public function with contract-violating arguments, which is out of scope. It also depends on `elixir_erl_clauses:guards/4` returning single-test alternatives when `Extra == []` (`elixir_erl_clauses.erl:40-48`), which makes `translate_guards/1` total for its actual inputs; verified by reading that function.
- No `binary_to_term`, no untrusted deserialization, and no OTP secure-coding "unsafe named function" call sites exist in this file, so the always-flag rule produced nothing.

### Ruled out
- **Code execution (Erlang-abstract-format generation).**
  Reason: the module emits abstract forms deterministically from the validated AST shape; it never evaluates external program text, and generated atoms/calls are all source literals.
  Axes examined: enum vs bitstring generators, guard alternatives, filters, `into`/`uniq`/`reduce` options, `%MapSet{}` literal, bound/unbound bitstring sizes | Not examined: none.
  Basis: verified.
- **Validation predicates (`valid_var_size/2`, `no_unbound_var/3`, `is_var/1`, `inline_or_into/1`, `returns_boolean/1`).**
  Reason: these only gate *which* (equivalent) lowering is chosen; a wrong answer either falls back to the general reduce path or yields Erlang that fails its own compile (unbound var) — neither is exploitable, and I confirmed both the valid and `error` size-var paths produce correct output at runtime.
  Axes examined: variable size, `default` size, integer/literal size, `op` (nested arithmetic) size, unbound pattern-local size var, string-literal segment expansion | Not examined: none.
  Basis: verified.
- **Resource consumption / recursion depth (`translate_gen`, `build_reduce_each`, `join_filters`, `collect_filters`, `no_unbound_var`).**
  Reason: recursion depth equals the count of generators/filters/segments written in the source, i.e. linear in source length with no amplification; the generated runtime code's cost tracks the comprehension normally.
  Axes examined: many generators, interleaved filters, many bitstring segments, string-literal expansion (N wildcards per N chars) | Not examined: none.
  Basis: verified.
- **Dead / unreachable defensive code (`comprehension_kind/1` `false` clause, the badarg catch-all clauses, `no_unbound_var` `error` branch, `?empty_map_set_pattern`).**
  Reason: each clause is reachable — `comprehension_kind(false)`/`(bin,[])`/`(nil,_)` all correspond to real `into` states, the runtime badarg clauses fire on ill-typed accumulators, the size-var `error` branch fired in my test #7, and the MapSet pattern fired in test #5.
  Axes examined: `into: false`/`%{}`/binary/MapSet, unbound-size fallback, badarg accumulator | Not examined: none.
  Basis: verified.
- **Shared mutable state / concurrency / process-dictionary leakage.**
  Reason: the module is purely functional — no `put`/`get`/`erase`, no ETS, no application env, no message passing, no registered-name resolution (grep returned zero matches for all of these).
  Axes examined: process dict, ETS, app-env, name races | Not examined: none.
  Basis: verified.
- **Deserialization / atom exhaustion / command / file / network / crypto / NIF sinks.**
  Reason: none of these primitives appear in the file (grep for the full OTP-secure-coding unsafe-call set returned no matches); all atoms are compile-time literals.
  Axes examined: `binary_to_term`, `*_to_atom`, `os:cmd`/`open_port`, `file:`/`filelib`, crypto, `load_nif` | Not examined: none.
  Basis: verified.

### Did I verify this?
Every ruled-out entry is `verified` with `Not examined: none`, so no `Depends on:` lines remain for the cross-file pass. Re-checking each:
- **Code execution / validation predicates / resource / dead-code / mutable-state** — upgraded beyond code-reading by executing all lowering branches (`ran /opt/elixir/bin/elixir for_test.exs`, 10/10 comprehension forms produced correct output with no crash), so the branch-reachability and semantic-equivalence claims rest on observed behavior, not inference.
- **Sink-absence (deserialization/atom/command/file/network/crypto/NIF)** — a negative existence claim; the search space is the whole file and the search is the grep over the complete unsafe-primitive set (zero matches), so the absence is substantiated rather than asserted.
- The one genuinely out-of-file dependency — that `elixir_expand.erl:854-879` guarantees the `{do, Expr}`/generator-start shape and `elixir_erl_clauses.erl:40-48` guarantees single-test guard alternatives — I resolved by reading those exact ranges in this pass, and the malformed-shape crash it would otherwise permit is reachable only by calling the internal `translate/3` with contract-violating args (out of scope). No unverified cross-file contract is being smuggled past the reader.

Evidence:
- read /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_for.erl:1-384 (whole file)
- read elixir_erl_pass.erl:188-189 (sole in-folder caller dispatches `{for,...}` here)
- read elixir_expand.erl:854-879 (expansion normalizes `for` AST → `{do,Expr}` head + validated generators/options)
- read elixir_erl_clauses.erl:40-48 (`guards/4` with `Extra=[]` yields single-test alternatives, making `translate_guards/1` total)
- read elixir_utils.erl:61-83,207-248 (`extract_guards`/`split_last`/`returns_boolean` shapes)
- ran `rg` over the file for dangerous sinks → "No matches found"
- ran `/opt/elixir/bin/elixir for_test.exs` → all 10 comprehension forms produced correct results (`guard+filter: [4]`, `into-mapset: MapSet.new([1, 2, 3])`, `bin-unbound-size: ["\t\t", "\a"]`, …) with no compile-time or runtime crash

