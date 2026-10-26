# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/behaviour.ex
<!-- sha256: c3b234325dec8adc2f9b0d14e9d9ac7817a3e4967ce41f1b209cd8b95f16339c -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:42:52Z_

I have enough to conclude. The file is byte-for-byte identical to the installed released Elixir (`diff … → IDENTICAL`), and I exercised every warning-producing code path through real compilation. Report follows.

## Summary
`Module.Behaviour` is a `@moduledoc false` compiler-internal module that checks `@behaviour`/`@impl` and formats the resulting diagnostics; every documented-shape warning path was driven through a real compile and produced correct, crash-free output, and the file is byte-identical to the released Elixir, so no genuine reachable bug was found.

## Bugfixes

No findings

### Ruled out

- **Warning-tuple ⇄ `format_warning/1` shape mismatch (arity crash in `format_diagnostic/1`).** The `check_impls` error path rewrites each `{:error, message}` via `message |> Tuple.insert_at(1, kind) |> Tuple.insert_at(1, fa)` (behaviour.ex:140), producing a 3–5-element tuple that a `format_warning` clause must accept. I enumerated all twelve emitted warning tags and confirmed a matching clause for each, then *verified* by compiling modules that trigger each one. Basis: `verified` — `elixir t2.exs` emitted correctly-formatted strings for `missing_callback` (`function bar/1 … not implemented`), `missing_impl`, `no_behaviours` (`got "@impl true" … but no behaviour was declared`), `impl_defined` (`… no behaviour specifies such callback. The known callbacks are:`), `impl_not_defined` (`got "@impl false" … but it is a callback specified in B1`), `behaviour_not_declared` (`got "@impl B2" … was not declared`), and `callback_not_defined` (`… B2 does not specify such callback`) — no `FunctionClauseError` from `format_warning`.
- **`Tuple.insert_at` index / arity arithmetic wrong (garbled `callback`/`kind` fields).** The double insert-at-1 must land `fa` then `kind` as elements 2 and 3. Verified indirectly: emitted messages show the correct pair and kind (`function foo/0`, `macro mac/1`, `function bar/1`), which could only be right if `fa` (with its user-facing arity) and `kind` were placed correctly. Basis: `verified`.
- **Macro arity off-by-one in `normalize_macro_or_function_callback/1` (behaviour.ex:286-295).** The `MACRO-` prefix strip with `arity - 1` must yield the user-facing macro arity. Verified: a `@macrocallback mac(any)` reported as `macro mac/1` (not `mac/2`), and a `defmacro` implementing a function callback / vice-versa were classified as `defmacro`/`def` consistently. Basis: `verified`.
- **`impls = for n <- (arity - defaults)..arity` range wrong for default args.** Ascending range for `defaults ≥ 0`; `defaults` never exceeds `arity` (optional args are a subset of formals). Callbacks with defaults matched correctly in testing. Basis: `verified` (bar/1 and foo/0 with `@impl true` matched and were popped, suppressing spurious `missing_callback`).
- **`callback_mismatch` not emitted when a callback carries a matching `@impl`.** A `defmacro foo` implementing a function-callback `foo` produced no mismatch warning, because `check_impls` `pop_in`s the callback (behaviour.ex:131) before `check_callbacks` runs. I confirmed this is the behaviour of the released compiler, not a deviation: the reviewed file is `IDENTICAL` (verified via `diff`) to `/opt/elixir/.../behaviour.ex`, so this is upstream-intentional (the `kind != wrong_kind` check at behaviour.ex:98 only applies to callbacks left unmatched by any `@impl`). Basis: `verified`.
- **Crash / `CaseClauseError` on any documented entry (`check_behaviours_and_impls/6`, `callbacks/1`, `format_diagnostic/1`).** Drove undeclared behaviours, non-behaviour modules, missing callbacks, missing `@impl`, `@impl true/false/Module` with every relevant combination through a full compile; all completed with only the expected warnings and no exception. Basis: `verified` (`elixir t.exs`, `elixir t2.exs`).

## Risks / caveats
- This module is entirely `@moduledoc false` / `@doc false` — a compiler-internal API with no *documented* public surface; per the scope rules the reportable surface is effectively empty, and I reviewed the compiler-callable functions (`check_behaviours_and_impls/6`, `callbacks/1`, `format_diagnostic/1`) as the practical entry points.
- The reviewed file is byte-identical to the installed released Elixir (`diff … → IDENTICAL`), so any defect here would be a latent upstream bug, not an injected one; my testing found none.
- I did not fuzz macro-generated `impl_context`/`:context` hygiene interactions in `warn_missing_impls`/`missing_impl_in_context?` beyond the hand-written cases above; that path is intricate but produced correct results in every case I drove, and it is unreachable except through the compiler's own `@impl` bookkeeping (not attacker-controlled input).

