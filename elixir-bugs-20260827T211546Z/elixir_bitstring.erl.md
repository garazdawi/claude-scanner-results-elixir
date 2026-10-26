# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_bitstring.erl
<!-- sha256: 0454e0b2b04674f1354cfbf4a4313825639bf78e49813d49299cc5fae852c250 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_bitstring.erl
<!-- sha256: 0454e0b2b04674f1354cfbf4a4313825639bf78e49813d49299cc5fae852c250 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

## Summary

Two internal-compiler-crash bugs, both triggerable from ordinary source and both surfacing as a raw Erlang error with an internal stack trace after (or instead of) the intended clean diagnostic:

1. An invalid integer/float size or unit spec makes `build_spec` return `[]`, which then hits an unguarded `[H | T] = build_spec(...)` match and crashes with a `MatchError` (high, verified empirically).
2. A bitstring specifier that is a **user macro** expanding to a malformed 3-tuple AST node (`{atom, meta, NonListNonAtom}`) reaches the incomplete `if` in `unpack_specs/2` (line 271) and crashes with an `if_clause` ErlangError. This was hedged as "out of scope / asserted" in the first pass; I have now reproduced it from a plain `.exs` file, so it is upgraded to a low-confidence finding.

## Bugfixes

#### `CSCAN-elixir_bitstring-build-spec-empty-badmatch`
- **Finding ID:** CSCAN-elixir_bitstring-build-spec-empty-badmatch
- **Severity:** High
- **Confidence:** high
- **Location:** `elixir_bitstring.erl:199`, `expand_specs/7` (interacting with the `[]`-returning branches of `build_spec/8` at lines 371–372 and 374–376)
- **Class:** Crash on stray / unexpected input (composite with State/accumulator: `function_error` no longer raises, so `[]` flows downstream)
- **User-visible outcome:** When a user writes a bitstring with an invalid float size (`<<1.0::float-size(8)>>`) or an integer/float unit without a size (`<<x::integer-unit(4)>>`), the compiler first prints the intended clean diagnostic ("float requires size*unit to be 16, 32, or 64…" / "integer and float types require a size specifier…") and then immediately aborts with an internal `** (MatchError) no match of right hand side value: []` plus an `:elixir_bitstring.expand_specs/7` Erlang stack trace. Instead of compilation failing gracefully with the friendly error (and continuing to collect further errors in the file), it dies with a raw internal compiler crash that looks like an Elixir bug and truncates error reporting.
- **Reproduction:**
  ```elixir
  defmodule T1 do
    def f, do: <<1.0::float-size(8)>>
  end
  ```
  Ran (see Evidence): emits the float-size error, then `** (MatchError) no match of right hand side value: []` at `src/elixir_bitstring.erl:199`. `build_spec/8` records the error via `function_error` and returns `[]`; because `function_error` taints the module and returns `ok` (it does not raise inside a module/function context), execution continues and `[H | T] = []` badmatches.
- **Suggested fix:** At line 199, guard the empty result instead of matching `[H | T]` unconditionally, e.g.
  ```erlang
  Expr = case build_spec(Meta, Size, Unit, MergedType, Endianness, Sign, MaybeInferredSize, ES) of
    [] -> {MergedType, Meta, nil};   %% error already recorded; module is tainted
    [H | T] -> lists:foldl(fun(I, Acc) -> {'-', Meta, [Acc, I]} end, H, T)
  end,
  {Expr, Alignment, SS, ES}.
  ```
  (Any benign placeholder spec works since the module is already tainted and will not be emitted.)
- **Risk of fix:** None observable. The module is already tainted, so compilation still fails with the intended diagnostic; the fix only removes the internal crash and lets error accumulation proceed. No successful-compile path reaches these `[]` branches, so no behavioural change for valid code.
- **Evidence:**
  - read `elixir_bitstring.erl:177-200` (the `[H | T] = build_spec(...)` match) and `elixir_bitstring.erl:340-388` (`build_spec/8` returning `[]` at lines 372 and 376).
  - ran `elixir t1.exs` → `error: float requires size*unit to be 16, 32, or 64 (default), got: 8` then `** (MatchError) no match of right hand side value: []`.
  - ran `elixir t2.exs` (`<<x::integer-unit(4)>>`) → `error: integer and float types require a size specifier if the unit specifier is given` then the same `MatchError`.

#### `CSCAN-elixir_bitstring-unpack-specs-if-clause` — Upgraded from hedged assumption in first pass
- **Finding ID:** CSCAN-elixir_bitstring-unpack-specs-if-clause
- **Severity:** Medium
- **Confidence:** low
- **Location:** `elixir_bitstring.erl:271`, `unpack_specs/2` clause at lines 270–272; reached from the macro-expansion path `expand_each_spec/6` lines 247/253.
- **Class:** Crash on stray / unexpected input (missing/incomplete clause: an `if` with no fall-through)
- **User-visible outcome:** A bitstring specifier that is a user-defined macro (`<<x::evil()>>`) whose expansion returns a malformed AST 3-tuple whose third element is neither an atom nor a list (e.g. `{:some_atom, [], 123}`) makes the compiler abort with a raw `** (ErlangError) Erlang error: :if_clause` at `src/elixir_bitstring.erl:271` plus an `:elixir_bitstring.unpack_specs/2` stack trace — no source location, no diagnostic. The neighbouring catch-all clause `unpack_specs(Other, Acc) -> [Other | Acc]` (lines 273–274) exists precisely so that unexpected shapes fall through to `expand_each_spec/6` line 256, which reports a clean `undefined_bittype` error; the line 270 clause pre-empts that catch-all for any `{atom, meta, _}` shape and then crashes on the `Args` shape inside the incomplete `if is_atom(Args) -> nil; is_list(Args) -> Args end`.
- **Why low confidence:** Triggering it requires a macro that emits genuinely malformed Elixir AST (the third element of an AST node is contractually an atom-context or an arg-list). That is macro misuse / a contract violation at the `elixir_expand`→`Macro.expand` boundary, not something the parser can emit directly. It is scored low because the input is exotic; it is nonetheless *source-reachable* (a plain `.exs` reproduces it) and the failure mode is a raw internal crash rather than the graceful `undefined_bittype` the surrounding code was written to produce, which is why it is a finding and not a dismissal.
- **Reproduction (ran, see Evidence):**
  ```elixir
  defmodule M do
    defmacro evil, do: {:some_atom, [], 123}
  end
  defmodule T do
    import M; require M
    def f(x), do: <<x::evil()>>
  end
  ```
- **Suggested fix:** Make the `if` total so a malformed third element falls through to the graceful path, e.g. treat any non-list `Args` as `nil` (matching how a bare atom context is handled) or drop to the catch-all:
  ```erlang
  unpack_specs({Expr, Meta, Args}, Acc) when is_atom(Expr) ->
    ListArgs = if is_list(Args) -> Args; true -> nil end,
    [{Expr, Meta, ListArgs} | Acc];
  ```
  This routes the malformed node to `validate_spec`/`expand_each_spec`, which report `undefined_bittype` cleanly instead of crashing.
- **Risk of fix:** Low. Well-formed input always has `Args` as an atom (context) or a list, both already handled identically; the change only affects the previously-crashing malformed case.
- **Evidence:**
  - read `elixir_bitstring.erl:262-274` (the `unpack_specs/2` clauses, incl. the incomplete `if` at 271 and the catch-all at 273–274) and `:247-258` (the `Macro.expand` → `unpack_specs(NewTypes, [])` re-entry that feeds unvalidated macro output back in).
  - ran `elixir mac.exs` (the reproduction above) → `** (ErlangError) Erlang error: :if_clause … src/elixir_bitstring.erl:271: :elixir_bitstring.unpack_specs/2 … :253: expand_each_spec/6 … :185: expand_specs/7 … :48: expand/8 … :25: expand/5`.

## Risks / caveats

- The `build_spec`-empty bug is reached from the documented public entry point `expand/5` (called by `elixir_expand` when compiling any module body). It requires a module/function compile context; in a bare eval/top-level context `function_error` falls through to `file_error`→`compile_error` and raises cleanly *before* line 199, so the crash is masked there. This is why that finding is scoped to module compilation.
- The `unpack_specs` `if_clause` crash does **not** depend on the taint/eval distinction — it is a hard `if_clause` inside `unpack_specs`, so it fires the same way in module and eval contexts as long as the malformed macro output reaches line 253.

### Ruled out

- **Operator / precedence typo** — Reason: guards (`Alignment rem 8 == 0`, `Type == binary; Type == bitstring`, `Size /= default; Unit /= default`) read correctly under explicit precedence; `;` alternatives and `andalso`/`orelse` in `expand_expr/4` match intent. Axes examined: each guard clause in `compute_alignment`, `type`, `size_and_unit`, `build_spec`, `maybe_warn_binary_alignment` | Not examined: none. Basis: verified.
- **Missing clause / catch-all** — Reason: `type/4`, `compute_alignment/3`, `validate_spec/2`, `expr_type/1`, `expand_each_spec/6` all have catch-all clauses. The two real gaps are (a) the `[]` case at line 199 and (b) the incomplete `if` in `unpack_specs/2` at line 271 — both promoted to findings above. Axes examined: default/integer/float/binary/bitstring/utf types, unknown spec atoms, macro-generated `{atom, meta, non-list}` third element (executed) | Not examined: none. Basis: verified.
- **Wrong atom / wrong symbol** — Reason: spec/type atoms (`size`, `unit`, `endianness`, `sign`, `type`, `binary`, `bitstring`, `utf8/16/32`) are consistent between `validate_spec/2`, the `Default` map, and `build_spec`. Axes examined: every atom key in the Default map vs. consumers | Not examined: none. Basis: verified.
- **State / accumulator threading error** — Reason: `BeforeVars`/`AfterVars` and `{S, OriginalS}` pairs are threaded deliberately (left-pattern vars excluded from right specs, restored after); `prematch` save/restore in `expand_spec_arg/4` is balanced. Axes examined: var-scope save/restore across `::` clause, prematch restore, alignment fold | Not examined: none. Basis: verified.
- **Off-by-one / wrong arithmetic** — Reason: `(Left+Right) rem 8`, `Size*Unit`, `number_size` and float-size checks are arithmetically correct; per-segment alignment is fully reduced mod 8 only at combination time, which is equivalent. Axes examined: size×unit products, alignment mod-8, float sizes 16/32/64 vs 8 | Not examined: negative size literals (rejected earlier by parser/spec-arg validation). Basis: verified.
- **Contract violation** — Reason: `expand/5` returns the documented `{'<<>>', Meta, Args}, S, E` triple on all paths; `validate_spec/2` returns `{Key, Val}` or `none`; `format_error/1` returns iolist for every atom this module emits. Axes examined: each return site of the three exports | Not examined: none. Basis: verified.
- **Resource leak on error path** — Reason: module is purely functional AST transformation; no files, ETS, processes, or monitors are created here. Axes examined: whole file for side-effecting BIFs | Not examined: none. Basis: verified.
- **Silent corruption** — Reason: the two crash paths produce a crash, not a silently-wrong spec; other paths emit well-formed specs. Axes examined: `concat_or_prepend_bitstring` reversal order, `add_spec`/`add_arg` ordering | Not examined: none. Basis: verified.
- **Infinite loop / hang** — Reason: all recursion (`expand/8`, `expand_each_spec/6`, `unpack_specs/2`) strictly consumes list structure; `Macro.expand` re-expansion at line 247 terminates because a fixpoint (`HA -> undefined_bittype` error) is detected. Axes examined: macro-expansion fixpoint, list recursion decrement | Not examined: a macro that expands to a *different* infinite chain of specifiers each step (would loop, but that is macro misuse producing new terms, out of scope). Basis: verified for source-level input.
- **Quadratic-where-linear** — Reason: accumulators use `[X | Acc]` with a single `lists:reverse` (`expand/8`), and `lists:reverse(Parts, Acc)` for concatenation; no `++` in loops. Axes examined: all accumulator appends | Not examined: none. Basis: verified.
- **Repeated work / Dead code / Race condition** — Reason: no compiled patterns, no shared mutable state, no per-call rebuilding; the `[]` branches were formerly dead (when `function_error` raised) but are now live and cause the reported crash. Axes examined: compile-pattern usage, process/global state | Not examined: none. Basis: verified.
- **Wrong scope / order of operations** — Reason: errors are recorded before continuing, and validations (`validate_expr`, `validate_size_required`) run before spec construction as intended. Axes examined: ordering of validate vs build | Not examined: none. Basis: verified.
- **Encoding / unicode mishandling** — Reason: no byte/codepoint handling here; utf types only tag the spec, actual encoding is downstream in `elixir_erl`. Axes examined: utf8/16/32 spec paths | Not examined: none. Basis: verified.
- **Documented behaviour mismatch** — Reason: no `@doc`/prose contract in this Erlang module beyond `-spec`s, which the exports satisfy. Axes examined: the three exports' specs | Not examined: none. Basis: verified.

### Did I verify this?

- The `build_spec`-empty finding is `verified` empirically (two `elixir` runs reproducing the `MatchError` at line 199), plus reads of `elixir_bitstring.erl` establishing that `function_error` returns `ok` in module context so `[]` flows downstream.
- The `unpack_specs` `if_clause` finding is `verified reproducible`: I constructed and ran `mac.exs`, a self-contained `.exs` defining a macro that expands a bitstring specifier to `{:some_atom, [], 123}`, and observed the `:if_clause` ErlangError at `unpack_specs/2:271` with the full `expand_each_spec/6:253 → expand_specs/7:185 → expand/8:48 → expand/5:25` stack. Its confidence stays **low** only because the *input* (a macro emitting malformed AST) is exotic, not because the crash is uncertain. This closes the first-pass "did not exhaustively fuzz `unpack_specs`" hedge and the `Not examined: macro-generated {foo, meta, 123}` axis — the axis was closed by *executing* the value through the real implementation, not by reading a neighbour.
- All `Basis: verified` ruled-out classes had their `Not examined` list closed to `none` (or to inputs the parser/spec-arg validation demonstrably rejects earlier) by reading the specific clauses and tracing source-level triggers; the "Missing clause / catch-all" axis for macro-generated malformed third elements was closed by execution.

### Upgraded from hedged assumption in first pass

- The first-pass **Ruled out → "Unclassified (malformed-AST `if_clause` in `unpack_specs/2`)"** entry (Basis: `asserted`, `Not examined: macro-generated {foo, meta, 123}`) is **upgraded to the finding `CSCAN-elixir_bitstring-unpack-specs-if-clause`** above. The prior pass declined to construct the trigger and called it out of scope. I constructed it and ran it: a user macro expanding a bitstring specifier to a malformed 3-tuple reproduces the `if_clause` crash from ordinary compilation. The corresponding first-pass Risks/caveats hedge ("I did not exhaustively fuzz the tokenizer-adjacent AST-shape assumptions in `unpack_specs/2`") is therefore removed and replaced by the executed reproduction.

### Dropped in re-review

- None. Both prior-report positions are preserved: the high-severity `build_spec`-empty finding is retained unchanged, and the previously-dismissed `unpack_specs` item is retained but moved from Ruled-out to a finding rather than dropped.
