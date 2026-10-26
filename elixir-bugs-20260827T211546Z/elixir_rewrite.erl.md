# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_rewrite.erl
<!-- sha256: 79507cca5bc87d2d590eb2997c8f9d89772afe9cef01b65f1dcb60383a0d192b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T06:30:07Z_

# Bug-hunting review — `elixir_rewrite.erl`

## Summary
The `guard/6` whitelist special-cases `Kernel.put_elem` and rewrites it to `erlang:setelement/3`, which is not a valid guard BIF, so a `put_elem` call in a guard is accepted here and fails downstream instead of being cleanly rejected.

## Spec-totality checklist
This file declares **no `-spec` attributes**, so there is no non-builtin spec surface to audit. Export-by-export reachability:
- `erl_to_ex/3` — exported, not called anywhere under `src`; consumer is out-of-folder (`.ex` lib, unreachable from here). Two return shapes by design (4-tuple for list arg, 2-tuple/`false` for integer arity). Covered.
- `inline/3` — called by `elixir_dispatch` (4 sites); thin wrapper over `inner_inline`, total via `inner_inline(_,_,_,_) -> false`. Covered.
- `rewrite/5` — called by `elixir_expand:988`; has catch-all clause. Covered.
- `match/6` — called by `elixir_expand:998` (Context=match); has catch-all. Covered.
- `guard/6` — called by `elixir_expand:998` (Context=guard); see finding below. Covered.
- `format_error/1` — three known error tuples; matches shapes produced by `match`/`guard`. Covered.

## Bugfixes

#### `CSCAN-elixir_rewrite-guard-put-elem-setelement`
- **Finding ID:** `CSCAN-elixir_rewrite-guard-put-elem-setelement`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `elixir_rewrite.erl:321-322`, `guard/6` clause for `put_elem`
- **Class:** Contract violation (guard-BIF whitelist bypass)
- **User-visible outcome:** A guard that reaches this clause (e.g. `when :erlang.setelement`-equivalent via `Kernel.put_elem/3`) is rewritten to `erlang:setelement/3` and returned as `{ok, …}` without any `allowed_guard/2` check. `setelement/3` is **not** an Erlang guard BIF, so instead of the clean Elixir diagnostic `cannot invoke … inside a guard` that every other non-guard-safe function receives (via the `guard(Receiver, …)` fall-through → `allowed_guard` → `{error, invalid_guard}`), the code is emitted into the guard and rejected much later with an opaque Erlang-level "illegal in guard" error pointing at compiler-generated code. The incorrect behaviour a caller notices: `put_elem` in a guard yields a confusing/misplaced compile error rather than Elixir's normal "you cannot invoke this inside a guard" message (and the front-end is effectively declaring `put_elem` guard-legal when it is not).
- **Reproduction:** In guard position write a remote `Kernel.put_elem(t, 0, x)` (or otherwise route `?kernel put_elem/3` through `expand_remote` with `Context =:= guard`, `elixir_expand.erl:998`). `guard(?kernel, _, put_elem, _, [t,0,x], S)` matches line 321 and returns `{ok, {{'.',_,[erlang,setelement]}, _, [increment(...), t, x]}}`. Contrast `elem` (line 319) → `erlang:element`, which *is* a guard BIF and is correct. A naïve attempt via a local `put_elem` may be gated earlier by dispatch; the explicit remote form reaches this clause.
- **Suggested fix:** Remove the `guard(?kernel, DotMeta, put_elem, …)` clause (lines 321-322) so `put_elem` falls through to the generic `guard(Receiver, …)` path; `allowed_guard(setelement, 3)` is not a guard BIF, producing the standard `{error, {invalid_guard, …}}`. (`is_map_key` and `elem` clauses are correct and should stay — both map to real guard BIFs.)
- **Risk of fix:** If any code legitimately relies on `put_elem` being usable in guards, that usage would now be rejected earlier with the standard Elixir error instead of the current late Erlang error — but since `setelement` cannot compile in a guard regardless, no currently-*working* program changes behaviour. Error message/shape for this specific misuse changes.
- **Evidence:** read `elixir_rewrite.erl:317-340` (three special guard clauses + `allowed_guard/2`); read `elixir_expand.erl:967-1003` (guard-context dispatch into `elixir_rewrite:Context/6`). `setelement/3` is not among Erlang's guard-allowed BIFs (`erl_internal:guard_bif/2`), unlike `element/2`.

#### `CSCAN-elixir_rewrite-reverse-fun-drops-index-arith`
- **Finding ID:** `CSCAN-elixir_rewrite-reverse-fun-drops-index-arith`
- **Severity:** Low
- **Confidence:** low
- **Location:** `elixir_rewrite.erl:269-276`, `inner_rewrite(erl_to_ex, …, element|setelement, …)`
- **Class:** Contract violation (returned reconstruction fun inconsistent with returned args)
- **User-visible outcome:** For the numeric-index `element`/`setelement` reverse rules, the eagerly-computed `ExArgs` applies the 1-based→0-based adjustment (`[Tuple, Index - 1]`), but the 4th-element reconstruction fun `fun([Index, Tuple]) -> [Tuple, Index] end` returns the index **without** the `-1`. For the `Index+1`-AST clauses (273-276) the fun's pattern `[Index, Tuple]` binds `Index` to the whole `{erlang,'+',[Index,1]}` AST node, again disagreeing with `ExArgs` (`[Tuple, Index]`). Any consumer that applies the reconstruction fun (rather than using `ExArgs`) to erlang args would get an off-by-one / un-normalized elixir index. Whether a caller notices depends entirely on out-of-folder code, which I could not reach.
- **Reproduction:** `erl_to_ex(erlang, element, [3, Tuple])` → `{'Elixir.Kernel', elem, [Tuple, 2], Fun}` where `Fun([3, Tuple])` returns `[Tuple, 3]`, not `[Tuple, 2]`.
- **Suggested fix:** Make the reconstruction fun mirror `ExArgs`: `fun([Index, Tuple]) -> [Tuple, Index - 1] end` for lines 269-272, and for 273-276 destructure the `+1` AST inside the fun. Only apply if a consumer actually uses the 4th element for these clauses.
- **Risk of fix:** Could change behaviour of whatever consumes the reconstruction fun; safe if no consumer uses it for these clauses (all four clauses share the same omission, which may be deliberate best-effort for non-round-trippable rewrites).
- **Evidence:** read `elixir_rewrite.erl:41-54, 260-283` (macro-generated reverse funs are consistent with their `ExArgs`; only the arithmetic clauses diverge). Consumer of `erl_to_ex/3` is not present under `src` (grep over `elixir/lib/elixir/src` shows no caller); reachability/impact unverifiable within scope → confidence low.

## Risks / caveats
- `erl_to_ex/3`'s only in-tree references are within this module; its real consumers live in the `.ex` standard library, which is outside the permitted read root (`/…/elixir/lib` was not accessible). The second finding's user-visible impact therefore could not be confirmed and is reported at low confidence deliberately rather than dismissed.
- The `put_elem` finding assumes `setelement/3` is not a guard BIF (well-established Erlang fact) and that `expand_remote` routes guard-context remote `?kernel:put_elem/3` calls into `guard/6` without an earlier gate (confirmed at `elixir_expand.erl:991-1002`). If upstream Elixir intentionally exposes `put_elem` in guards, this is still a defect because the emitted `setelement` cannot compile in a guard.

### Ruled out
- **Operator / precedence typo** — Reason: guards here are single simple tests (`is_number`, `is_list`, `is_integer`); no mixed `,`/`;` or `andalso`/`orelse` compound guard exists to mis-partition. Axes examined: `is_number(Index)` with integer, float, non-number; `is_list(Tail)` with proper/improper/non-list | Not examined: none. Basis: verified.
- **Missing clause / catch-all** — Reason: every recursive/dispatch function has a terminal catch-all (`inner_inline/4`:241, `inner_rewrite`:282-283, `rewrite/5`:256, `match/6`:301, `static_append`:310, `is_always_string`:371). Axes examined: unknown Mod/Fun/Arity, non-erlang receiver in guard, arity mismatch for elem/put_elem/is_map_key, non-list/binary/var Left in `++` match | Not examined: none. Basis: verified.
- **Wrong atom / wrong symbol** — Reason: `?kernel`/`erlang` module atoms and BIF names match their Erlang counterparts across inline/rewrite tables; `catch impossible ->` correctly catches the `throw(impossible)`. Axes examined: inline pairs spot-checked (`is_map_key` arg swap, `elem`/`element` index base, bitwise ops), throw class vs catch pattern | Not examined: exhaustive re-verification of all ~130 inline rows against OTP. Basis: asserted. Depends on: OTP `erlang`/`maps`/`binary`/`os` BIF names being stable as written.
- **State / accumulator threading error** — Reason: module is stateless transformation; `_S` scope is only threaded to `elixir_utils:guard_info(S)` for error text, never mutated. Axes examined: guard/match error paths' use of `S` | Not examined: none. Basis: verified.
- **Off-by-one / wrong arithmetic** — Reason: `elem`↔`element` index conversion (`increment`/`Index-1`) is correct 0-vs-1 base; the reverse-fun omission is captured as its own finding. Axes examined: numeric index, `Index+1` AST form, `increment` on number vs non-number | Not examined: none. Basis: verified.
- **Contract violation** — Reason: reported as findings above (put_elem guard, reverse-fun); other return shapes match callers' `case` patterns in `elixir_expand`. Axes examined: `{ok,_}`/`{error,_}` from guard/match, 2- vs 4-tuple from erl_to_ex | Not examined: none. Basis: verified.
- **Crash on stray / unexpected input** — Reason: catch-alls prevent `function_clause`; `match/6` non-list/binary/var `Left` routes to `throw(impossible)`→`{error,_}`; `format_error`'s `Receiver:'__info__'` is wrapped in `catch _:_`. Axes examined: var, binary, improper-list, charlist, proper-list Left; non-Elixir receiver (`erlang`) in format_error | Not examined: none. Basis: verified.
- **Resource leak on error path** — Reason: no files, ETS, processes, or monitors are created in this module. Axes examined: whole-file scan | Not examined: none. Basis: verified.
- **Silent corruption** — Reason: `is_always_string/3` whitelist entries (`Enum.join`, `Enum.map_join`, `Kernel.inspect`, `Macro.to_string`, `String.Chars.to_string`, `Path.join`) all return binaries, so skipping the `to_string` wrapper is sound. Axes examined: each listed Module/Function's return type; non-atom Module AST (→false); binary literal Arg (→identity) | Not examined: none. Basis: verified.
- **Infinite loop / hang** — Reason: `static_append`/`inner_*` recursions strictly consume list structure; no `receive`. Axes examined: nested `{'|',...}`, multi-element, single-element list recursion termination | Not examined: none. Basis: verified.
- **Quadratic-where-linear** — Reason: list building uses `[H | recurse(T)]`, not `Acc ++ X`; no per-call pattern recompilation. Axes examined: `static_append` construction, absence of `++`/`compile_pattern` in loops | Not examined: none. Basis: verified.
- **Repeated work** — Reason: inline/rewrite tables are compiled clause heads (no per-call list rebuilds). Axes examined: `inner_inline`/`inner_rewrite` dispatch | Not examined: none. Basis: verified.
- **Dead / unreachable code** — Reason: all clauses reachable from `inline`/`rewrite`/`match`/`guard`/`erl_to_ex`; `erl_to_ex/3` itself has no in-`src` caller but is an exported public API (consumer in `.ex` lib, out of read scope). Axes examined: caller grep over `src` | Not examined: `.ex` callers (path outside read root). Basis: asserted. Depends on: `elixir/lib/**/*.ex` invoking `:elixir_rewrite.erl_to_ex/3`.
- **Documented behaviour mismatch** — Reason: module comments describe inline (same arity/order) and rewrite (may change arity, not order) semantics; observed clauses conform (arg order preserved; only arity/index adjusted). Axes examined: each rewrite clause's arg-order preservation | Not examined: none. Basis: verified.
- **Race condition / shared mutable state** — Reason: pure functions, no shared state or process interaction. Axes examined: whole-file | Not examined: none. Basis: verified.
- **Wrong scope / order of operations** — Reason: no side effects; ordering of clauses (numeric `is_number` element before `+1`-AST element) is correct and non-overlapping. Axes examined: clause ordering 269 vs 273 | Not examined: none. Basis: verified.
- **Encoding / unicode mishandling** — Reason: module manipulates AST terms/atoms, never decodes bytes vs codepoints. Axes examined: whole-file | Not examined: none. Basis: verified.

### Did I verify this?
- Every entry marked **verified** had its `Not examined` list driven to `none` by tracing the actual clauses against the enumerated input axes for the datatype in front of me (AST lists/tuples, atoms, numbers) — e.g. I traced `static_append` on var/binary/improper/charlist/proper Left, and `guard`/`match` on arity mismatches and non-erlang receivers.
- **Wrong atom / wrong symbol** remains **asserted**: I spot-checked the semantically interesting rows (arg swaps, index base) but did not re-verify all ~130 inline mappings against an OTP reference; upgrading requires cross-checking each `?inline` row against `erlang`/`maps`/`binary`/`os` exports. Depends on: OTP BIF-name stability.
- **Dead / unreachable code** remains **asserted** for `erl_to_ex/3` reachability: the `.ex` caller tree is outside the permitted read root, so I could not confirm the live consumer. Depends on: `elixir/lib/**/*.ex` calling `:elixir_rewrite.erl_to_ex/3`. This same inaccessibility is why the `reverse-fun` finding is filed at low confidence rather than dismissed.

