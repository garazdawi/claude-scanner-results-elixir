# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/macro/env.ex
<!-- sha256: 8dd7594b076d33ba3a91596697dcd14650b4c45257a3208de8c851022ac12801 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/macro/env.ex
<!-- sha256: 8dd7594b076d33ba3a91596697dcd14650b4c45257a3208de8c851022ac12801 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T00:00:00Z (hedge-elimination pass over the 2026-08-27 first pass)_

I re-ran every hedged claim as an executed experiment against the compiled reference build (Elixir 1.21.0-dev / OTP 29) instead of reasoning from the neighbour source. Reading `:elixir_aliases`/`:elixir_dispatch` would only tell me what their guards *are*, not which inputs they disagree with `env.ex` on, so I pushed concrete values through the real public functions and observed the outcomes. Two crash paths that the first pass dismissed "from reading" turned out to be reachable; both are promoted to low-confidence findings below.

## Summary
The documented happy-path behaviour is correct (16/16 doctests still pass). Trying the previously-unexamined **input** axes (as opposed to the delegate **return** axes the first pass looked at) surfaced two reachable crash-on-input paths in the public API:

1. `expand_alias/4` raises `FunctionClauseError` on the empty segment list `[]`, which is admitted by both the `is_list/1` guard and the `[atom()]` spec.
2. `expand_import/5` raises `BadBooleanError` (a cryptic, misleading message) when `allow_locals` is any non-boolean that is not a 0-arity function.

Both are low severity (compiler-integration surface used by tools/language servers, not attacker-facing at runtime), but both are genuine reachable crashes on inputs the type spec/guard permit, so I am recording them rather than leaving them as ruled-out assertions.

## Findings

### 1. `expand_alias/4` crashes with `FunctionClauseError` on the empty segment list `[]`
**Upgraded from hedged assumption in first pass** (the first-pass entry examined only what `:elixir_aliases.expand` *returns*, never what happens when `[]` is passed *in*).

- **Location:** `lib/macro/env.ex:531-538`
- **Confidence:** Low
- **Sink class:** crash-on-stray-input / contract-boundary (public API admits an input its delegate rejects with a raw `FunctionClauseError`).
- **Detail:** The head guard is `when is_list(meta) and is_list(list) and is_list(opts)` (line 532) and the `@spec` types `list` as `[atom()]` (line 529) — both admit `[]`. The body calls `:elixir_aliases.expand(meta, list, env, trace)` (line 535), which has no clause for an empty list. The `case` at lines 535-538 (`atom` vs `[_ | _]`) is therefore never reached for this input; the crash happens *before* it, inside the delegate.
- **Executed evidence:**
  ```
  Macro.Env.expand_alias(__ENV__, [], [])
  #=> ** (FunctionClauseError) no function clause matching in :elixir_aliases.expand/5
  ```
  For contrast, spec-conforming non-empty inputs return cleanly: `[:Elixir]` → `:error`, `[Foo, :Bar]` → `:error`, `[:Foo, :Bar]` → `:error`.
- **Why low and not higher:** `[]` is semantically degenerate (an alias with no segments); no compiler path is known to pass it, and it is arguably a caller contract violation. But it is type-admissible and the failure mode is an opaque delegate `FunctionClauseError` rather than a controlled `:error`, so it is worth a downstream look. A one-line `[] -> :error` (or `[H | _]` guard on the head) would close it.

### 2. `expand_import/5` raises `BadBooleanError` on a non-boolean, non-0-arity `allow_locals`
**Upgraded from hedged assumption in first pass** (first pass: "no `BadBooleanError` on any spec-conforming input", basis "asserted from reading" — the non-conforming axis was never tried).

- **Location:** `lib/macro/env.ex:594-601` (specifically the `allow_locals and function_exported?(...)` at line 597)
- **Confidence:** Low
- **Sink class:** crash-on-stray-input / misleading-error (contract boundary — the offending value is outside the `boolean() | (-> function() | false)` spec).
- **Detail:** Line 594 peels off only the *0-arity* function case via `is_function(allow_locals, 0)`. Any other non-boolean — a function of different arity, or a plain atom — falls to line 597 where it becomes the left operand of `and`, raising `BadBooleanError`. There is no guard on `allow_locals` (the head guards only `name`, `arity`, `opts`).
- **Executed evidence:**
  ```
  Macro.Env.expand_import(__ENV__, [line: 1], :foo, 1, allow_locals: false)        #=> {:error, :not_found}   (ok)
  Macro.Env.expand_import(__ENV__, [line: 1], :foo, 1, allow_locals: fn -> false end) #=> {:error, :not_found} (ok)
  Macro.Env.expand_import(__ENV__, [line: 1], :foo, 1, allow_locals: fn _ -> false end)
    #=> ** (BadBooleanError) expected a boolean on left-side of "and", got: #Function<...>
  Macro.Env.expand_import(__ENV__, [line: 1], :foo, 1, allow_locals: :maybe)
    #=> ** (BadBooleanError) expected a boolean on left-side of "and", got: :maybe
  ```
- **Why low:** The crashing inputs are non-spec-conforming, so this is a robustness/diagnostics gap rather than a correctness bug on valid input. The concern is that the error (`BadBooleanError` deep in `and`) is opaque for a tool author who passed, e.g., a wrong-arity resolver fun; a clearer `FunctionClauseError`/argument check would be friendlier. Recorded at low confidence because the axis is now confirmed by execution.

## Ruled out

- **`fetch_alias/2` and `fetch_macro_alias/2` call `String.to_unsafe_atom/1`** (lines 216, 222) — not an injected undefined function. **Basis: verified.** `String.__info__(:functions)` lists `to_unsafe_atom: 1`; `Macro.Env.fetch_alias(env, :Bar)` returns `{:ok, Foo.Bar}`. Axes examined: existence-at-runtime, observable return. Not examined: none.

- **All `@doc` doctests** (`has_var?`, `lookup_import`, `lookup_alias_as`, `required?`, `define_require`, `define_import`, `define_alias`, `expand_alias`) — **Basis: verified.** Recompiled the file-under-review so it redefined `Macro.Env`, ran `doctest Macro.Env` → 16 passed. Axes examined: every documented example input. Not examined: none.

- **`expand_alias/4` `case` (`atom` vs `[_|_]`, lines 535-538) missing an `[]` *return* clause** — no `CaseClauseError` from a `[]` **return** of the delegate. **Basis: verified by execution.** For every *non-empty* input tried (`[:Elixir]`, `[:Foo, :Bar]`, `[Foo, :Bar]`, `[:MyList, :Nested]`), `:elixir_aliases.expand` returned either an atom or the original `[_|_]` list — never `[]` — so the `case` is total for reachable **returns**. Axes examined: alias-hit (atom return), `Elixir`-prefixed / non-alias (list return), nested. Not examined: none. *(The separate empty-list **input** axis is not ruled out — it is Finding 1 above.)*

- **`stacktrace/1` `cond` (lines 705-716) missing a branch** — no `CaseClauseError`; the `cond` ends in `true ->` and is therefore total over `{module, function}` presence. **Basis: verified by execution.** Axes examined: module=nil (→`:elixir_compiler`/`:__FILE__`), module set & function=nil (→`:__MODULE__`/0), both set (→`{name,arity}`), and module=nil with function set (module-nil branch wins, as expected). Not examined: none of the presence-combinations. **Note:** a *malformed* `env.function` (non-`nil`, non-2-tuple, e.g. `:weird`) raises `MatchError` at line 713 (`{name, arity} = env.function`) — I ran this; it is a `Macro.Env` struct-field contract violation (`function` is typed `name_arity | nil`), not reachable through the compiler, so it stays ruled out rather than becoming a finding.

- **`expand_import/5` spec-conforming `allow_locals` (boolean or 0-arity fun)** — no crash. **Basis: verified by execution** (`false` and `fn -> false end` both return `{:error, :not_found}`, see Finding 2 evidence block). Axes examined: `false`, 0-arity fun. Not examined: `true` with a real module having `__info__/1` (would exercise line 598's `[{module, module.__info__(:macros)}]`), which requires an `env` whose `module` is a compiled module — not constructed here. *(The non-conforming `allow_locals` axis is Finding 2.)*

- **`define_*` `@spec`s vs implementations** (`{:ok,t}|{:error,String.t()}` via `maybe_define_error/2`) — **Basis: verified by execution.** `Macro.Env.define_alias(__ENV__, [line: 1], :an_atom)` returned `{:error, "alias cannot be inferred automatically for module: :an_atom, ..."}` — the `{:error, String.t()}` shape the spec admits, matching the documented doctest string. `define_require`/`define_import` `{:ok, env}` returns are exercised by the passing doctests. Axes examined: error-shape (bad module) and ok-shape (doctests). Not examined: none for the return-shape claim.

- **`location/1`, `vars/1`, `in_guard?/1`, `in_match?/1`, `prune_compile_info/1`, `prepend_tracer/2`, `to_guard/1`, `to_match/1`** — pure struct reads/updates over typed fields; return shapes match their `@spec`s and are exercised by the doctest suite and struct semantics. **Basis: verified (doctests) / structural.** Axes examined: documented field shapes. Not examined: none material.

## Did I verify this?

- **Findings 1 & 2:** Verified by *execution* — the crashes above were produced by running the real public functions in the reference build, not inferred from delegate source. The residual uncertainty (why "Low") is about *reachability/severity in practice* (are `[]` segments / mistyped `allow_locals` ever passed by a real tool?), not about whether the crash occurs — that is confirmed.
- **Ruled-out entries:** Each now carries `Axes examined` / `Not examined`, and every "Not examined: none" is backed by inputs actually pushed through the function (listed inline), not by reading `:elixir_aliases`/`:elixir_dispatch`. The one remaining un-pushed axis (`expand_import` with `allow_locals: true` against a real compiled `env.module`) is stated explicitly rather than claimed clear.
- **Environment caveat:** The `latin1` VM-encoding warning was present in all runs; it does not affect the atom/boolean/list code paths exercised here (no non-ASCII data involved). I still could not diff against pristine upstream (no VCS), so "no injected mutation" continues to rest on doctest + spec + reference-build agreement, as in the first pass.

## Dropped in re-review

No prior findings were dropped. The first pass reported *no* findings; this pass preserves all of its ruled-out conclusions (now with executed axes) and *adds* the two findings above where trying the previously-unexamined input axes revealed reachable crashes.
