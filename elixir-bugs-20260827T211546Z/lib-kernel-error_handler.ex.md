# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/error_handler.ex
<!-- sha256: 8a001c513d47d859f60b4a0c8c9d8dc3b0a17156af944ec255f2fb839093b2f3 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:26:46Z_

This file is tiny and I've read it in full. Let me verify the one non-obvious semantic concern (the `or` with a non-boolean right operand) — already done above — and reason through the rest.

The key checks:
- `ensure_loaded/1` returns a strict `boolean` (both `:code.ensure_loaded` result shapes are matched), so it's a valid left operand for `or`. ✅
- `false or :found` returns `:found` without raising (verified above), and that return value is discarded — no crash. ✅
- The `receive` is total over `{^ref, _}` messages (specific `{:loading, pid}` clause first, then catch-all `{^ref, value}`). ✅
- The inner monitor fires `:DOWN` and is auto-removed — no monitor leak. ✅

# Review: `kernel/error_handler.ex`

- **Summary:** This 60-line Erlang `error_handler` shim (loading/compiling modules on undefined-function/lambda calls, coordinating with the parallel compiler) contains no genuine, reachable bugs.

## Bugfixes

No findings

## Risks / caveats

- The module is `@moduledoc false` and its "public" surface is really the Erlang `error_handler` behaviour callbacks (`undefined_function/3`, `undefined_lambda/3`), invoked by the BEAM, plus `ensure_compiled/4`/`ensure_loaded/1` called by `Kernel.ParallelCompiler`. I treated those as the documented entry points.
- The `receive` blocks in `ensure_compiled/2`(clause with 4 args) have no timeout. This is **by design**: the parallel-compiler protocol guarantees exactly one reply tagged with `ref`, and a hang here would reflect a compiler-coordination bug elsewhere, not this file. Not reported as a defect.
- Behaviour under an already-dead `pid` in the `{:loading, pid}` path relies on `:erlang.monitor/2` immediately delivering `:DOWN` for a dead process — standard OTP semantics — so the inner `receive` still returns `:found`. Not a bug.

### Ruled out

- **`ensure_loaded(module) or ensure_compiled(...)` crashing because `ensure_compiled` returns a non-boolean atom (`:found`/`:not_found`/`:deadlock`)** — `or/2` only requires its *left* operand to be a boolean; the right operand may be any term and is returned as-is. `ensure_loaded/1` always returns a boolean, and the `or` result is discarded. Basis: `verified` — `elixir -e 'IO.inspect(false or :found)'` printed `:found` with no error.
- **Non-total `receive` leaving compiler-reply messages stranded** — the two clauses `{^ref, {:loading, pid}}` and `{^ref, value}` cover every message tagged with the freshly-created `ref`; the catch-all binds `value` for all non-`:loading` replies (`:found`/`:not_found`/`:deadlock`). Basis: `asserted` (pattern-coverage reasoning over the two literal clauses in this file).
- **Monitor leak in the `{:loading, pid}` branch** — the `:erlang.monitor/2` reference is consumed by the following `receive {:DOWN, ^ref, ...}`, and OTP auto-removes a monitor once its `:DOWN` is delivered; no `demonitor` is needed. Basis: `asserted` (OTP monitor semantics; single `:DOWN` consumed in-line).
- **`ensure_compiled(nil, ...)` mismatch** — the `nil` module head returns `:not_found` before any compiler messaging, matching the spec's `:not_found` result and the "never wait on nil" comment. Basis: `asserted` (clause is present at line 32).
- **`@spec` disagreements** — all four specs match their implementations' argument arity/types and return shapes (`ensure_loaded :: boolean`, `ensure_compiled :: :found | :not_found | :deadlock`, the two callbacks `:: term`). Basis: `asserted` (direct read of each spec against its clause).

