# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/tuple.ex
<!-- sha256: 0d9ff60bedda35fb3c95c1eb85382efe8d4886acef1b71fc7a96c0f8cfb47711 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:46:51Z_

I now have a complete, verified picture of every function in the file. Writing the report.

---

## Summary

`Tuple` is a thin, self-contained wrapper over BEAM tuple built-ins with no security-relevant sinks — no code execution, atom creation, I/O, deserialization, dispatch, shared state, or Unicode handling — and every out-of-contract input produces a clean, documented exception.

## Security

`No findings`

Every public function (`duplicate/2`, `insert_at/3`, `delete_at/2`, `append/2` [deprecated], `sum/1`, `product/1`, `to_list/1`) delegates directly to an `:erlang` built-in. I walked the Phase 1 inventory against the file: none of the code-execution, compile-time-execution, atom-creation, command, file/path, archive, deserialization, template, dispatch, network, crypto, Unicode, shared-state, or concurrency classes have a sink in this file. The two classes with a plausible surface (resource consumption, dead defensive code) were traced and reproduced, and are ruled out below.

### Ruled out

- **`duplicate/2` allocation DoS (resource consumption).** `:erlang.make_tuple(size, data)` — cost scales with `size`. But a single call is hard-bounded by the BEAM tuple-arity limit: `Tuple.duplicate(:x, 16_777_215)` succeeds; `16_777_216`, `1_000_000_000`, and `-1` all raise a clean `ArgumentError` ("1st argument: out of range"). Worst-case single allocation is ~134 MB then a hard error, not unbounded growth — and allocating `size` slots is the function's documented purpose, identical in shape to `List.duplicate/2`/`String.duplicate/2`. Basis: **verified** (`elixir -e 'Tuple.duplicate(:x, 16_777_216)'` → `%ArgumentError{message: "...out of range..."}`; `16_777_215` → `size=16777215`).
- **`sum/1` and `product/1` crash on non-numeric tuples.** `acc + :erlang.element(...)` raises `ArithmeticError` on `{:a, :b}`. This requires the caller to pass a tuple that violates the `:: number()` contract (input should be a tuple of numbers); the result is a clean exception, not memory/logic corruption. Empty-tuple identities are correct (`sum({})`=0, `product({})`=1). The recursive clause is tail-recursive, so no stack growth over the (≤2^24-bounded) tuple. Basis: **verified** (`Tuple.sum({:a,:b})` → `ArithmeticError`; `Tuple.sum({})` → `0`).
- **`insert_at/3` / `delete_at/2` index handling.** Out-of-range and negative indices raise a clean `ArgumentError` (documented). Integers are arbitrary-precision on the BEAM, so `index + 1` cannot overflow. Basis: **verified** (`Tuple.insert_at({:a}, 5, :x)` → `ArgumentError`; `Tuple.delete_at({}, 0)` → `ArgumentError`).
- **Guards as dead defensive code.** These functions are "Inlined by the compiler," so at a direct call site the `size >= 0` / `index >= 0` / `is_integer` guards are bypassed (`Tuple.duplicate(:x, -1)` inlined → `ArgumentError` from `make_tuple`, not `FunctionClauseError`). This is not a hole: the underlying built-in validates identically, and on the non-inlined path (`apply/3`, function capture) the Elixir guard does fire (`apply(Tuple, :duplicate, [:x, -1])` → `FunctionClauseError`). Defense is redundant with the built-in, never load-bearing-but-absent. Basis: **verified** (both paths reproduced).

## Risks / caveats

- Scope was the single file `tuple.ex`. The security posture of the delegated BEAM primitives (`:erlang.make_tuple`, `insert_element`, `delete_element`, `append_element`, `tuple_to_list`, `element`) is assumed correct — they are ERTS C built-ins outside this repo and were exercised only through their observable Elixir behavior, not audited.
- Reproductions ran against the in-tree build (Elixir 1.21.0-dev / Erlang-OTP 29). The ~134 MB / 16,777,215-element tuple ceiling is a BEAM VM constant and could differ on other ERTS versions/architectures, but the qualitative result (bounded single allocation, clean error beyond) is structural.
- `sum/1` and `product/1` on a tuple mixing large integers/floats can produce large-magnitude or float results, but that is ordinary arithmetic, not a boundary this module is responsible for guarding; no finding.

