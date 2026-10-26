# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_dispatch.erl
<!-- sha256: f51f579e64a84c079af268323cd7e7677ed41ec04fa40b23ce6ba479c58bb378 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T06:07:11Z_

I've read the full file. Here is my review report.

## Summary

`elixir_dispatch.erl` faithfully implements import/require dispatch and macro expansion, and I found no correctness, crash, contract, or resource defect reachable through its documented public API.

## Bugfixes

No findings

## Risks / caveats

- The module is stateless library code invoked inside the compiler process; every export threads `E`/`S` records that are library-internal but well-typed. Several dismissals below depend on invariants guaranteed by neighbour modules (`elixir_import` producing sorted ordsets, `elixir_env`/`elixir_expand` supplying a normalized env map). Those cross-file assumptions are enumerated per-class and could not be discharged from this file alone.
- I did not execute code (no toolchain run); all reasoning is from reading the source.

### Ruled out

**Operator / precedence typo**
- Reason: The two guard-heavy sites (`check_deprecated/6` line 427, `check_deprecated/6` line 435) and the AllowLocals guard (line 114) all parenthesize every comparison feeding `or`/`orelse`, so the tighter-than-comparison binding of `or` cannot mis-associate them; the `,`-vs-`;` partition in `dispatch_import` (line 114) and `find_imports_by_name` (lines 296–307) matches the intended set.
- Axes examined: `Module=nil` vs atom, `Kind=macro` vs `function`, `function=nil` vs bound, `Name` in/out of the get_env/fetch_env set, self-referential `{N,A}` call vs different call | Not examined: none
- Basis: verified

**Missing clause / catch-all**
- Reason: `dispatch_import` (line 133) has an `Error` catch-all for `{ambiguous,_}`/`{conflict,_}`; `dispatch_require` covers atom and non-atom receivers (lines 143/163); `expand_import`/`do_expand_import` enumerate every Dispatch shape (`{ambiguous,_}`,`{import,_}`,`{function,_}`,`{macro,_}`,`false`); `is_macro`/`get_info` catch `error:_` around `__info__`.
- Axes examined: ambiguous, conflict, function-import, macro-import, not-found, non-atom receiver, module lacking `__info__` | Not examined: none
- Basis: verified

**Wrong atom / wrong symbol**
- Reason: Receiver skip-list atoms (`erlang`,`elixir_def`,`elixir_module`,`?kernel`,`?application`), the `get_env`/`fetch_env`/`fetch_env!` names, and `__info__(macros)`/`__info__(functions)`/`__info__(deprecated)` keys all match the callee contracts; `macro_name`/`Arity+1` pairing is consistent between `expander_macro_named` and `expand_macro_fun`.
- Axes examined: each skip atom, each env-fn name, each `__info__` key | Not examined: none
- Basis: verified

**State / accumulator threading error**
- Reason: `find_imports/3` threads `Acc0→Acc1→Acc2` monotonically; `find_imports_by_name` returns `NewAcc` correctly; `expand_import` computes `Local` once and reuses it; no stale `St0`/`St1` reuse exists (this file passes `S`/`E` through unchanged into `expand_quoted`).
- Axes examined: empty imports map, single-arity collision, multi-module accumulation | Not examined: none
- Basis: verified

**Off-by-one / wrong arithmetic**
- Reason: The only arithmetic is `Arity + 1` for macro fun arity, applied identically at `expander_macro_named` (line 257), `expand_macro_fun` MFA (line 267), and `expand_quoted` MFA (line 280); `apply(Fun,[Caller|Args])` supplies exactly `length(Args)+1` args.
- Axes examined: arity 0, arity N | Not examined: none
- Basis: verified

**Contract violation**
- Reason: `remote_function` always returns a 4-tuple `{remote,_,_,_}`; `expand_import` returns only shapes `dispatch_import` handles; `expand_require/6,7` return `{macro,_,_}` or `error`; `find_imports/3` returns a sorted proplist. All match caller expectations in `import_function`/`dispatch_*`.
- Axes examined: inline-hit vs miss, macro vs function vs local vs not_found | Not examined: none
- Basis: verified

**Crash on stray / unexpected input**
- Reason: Public entry points take `Meta` (proplist), `Name`/`Arity`, and the env map; `is_import`/`is_element`/`lists:keyfind` tolerate missing meta keys (return `false`); `is_macro` guards `__info__` failures. Inputs violating the `Meta`/env `-spec` are excluded per scope.
- Axes examined: meta without `imports`/`context`, unloaded receiver module, receiver without `__info__` | Not examined: none
- Basis: verified

**Resource leak on error path**
- Reason: The module opens no files/ETS/processes/monitors; `code:ensure_loaded` in `get_deprecations` acquires nothing the caller must release.
- Axes examined: ensure_loaded success vs `{error,_}` | Not examined: none
- Basis: verified

**Silent corruption**
- Reason: Ambiguity, conflict, and inline-rewrite paths each produce a distinct, downstream-matched shape; `stop_generated` only prepends metadata to list-meta triples and passes everything else through unchanged.
- Axes examined: triple with list meta, non-triple element, inline rewrite hit | Not examined: none
- Basis: verified

**Infinite loop / hang**
- Reason: No `receive`/gen_server call in this file; `prune_stacktrace` and `find_imports_by_name` recurse over strictly shrinking lists.
- Axes examined: empty list, long list | Not examined: none
- Basis: verified

**Quadratic-where-linear / Repeated work**
- Reason: List accumulation uses map inserts (`Acc#{Arity => Mod}`) and `[H | prune_stacktrace(...)]`, not `++` in a loop; `find_imports_by_name` short-circuits on sorted order; no pattern is recompiled per call.
- Axes examined: many imports, deep stacktrace | Not examined: none
- Basis: verified

**Dead / unreachable code**
- Reason: Every `expand_import`/`do_expand_import` branch is reachable from `dispatch_import`/`import_function`; the `Fun when is_function(Fun,0)` AllowLocals branch is a documented external-caller feature.
- Axes examined: AllowLocals = false/true/fun | Not examined: none
- Basis: verified

**Documented behaviour mismatch**
- Reason: The module comment ("assumes scope info is normalized ordsets") is honoured via `ordsets:is_element`; `check_deprecated`'s comment ("compile-time behavior cannot be verified by the runtime group pass") matches the `function==nil or Kind==macro` gate.
- Axes examined: function body vs module body, macro vs function kind | Not examined: none
- Basis: verified

**Race condition / shared mutable state**
- Reason: No shared mutable state is touched here; env/scope are per-process values.
- Axes examined: n/a (no shared state) | Not examined: none
- Basis: verified

**Wrong scope / order of operations**
- Reason: `check_deprecated` is invoked before `Callback` in both inline and non-inline branches consistently; `Local` is computed before the conflict/dispatch decision it feeds.
- Axes examined: inline path, non-inline path, conflict path | Not examined: none
- Basis: verified

**Encoding / unicode mishandling**
- Reason: This file manipulates atoms and `{Name,Arity}` tuples only; no byte/codepoint handling occurs (that lives in the tokenizer/interpolation modules).
- Axes examined: n/a | Not examined: none
- Basis: verified

**Unclassified**
- Reason: No residual file-specific anomaly surfaced after reading all 448 lines and tracing each export's return shape against its in-folder callers.
- Axes examined: all exports' return/contract shapes | Not examined: none
- Basis: verified

### Did I verify this?

Every ruled-out entry above is Basis `verified`: I read the full file (`read elixir_dispatch.erl:1-448`), traced each trigger from a documented export, and each `Not examined` list reads `none`, so no `Depends on:` lines remain — with two scoped exceptions I make explicit here rather than hide:

- The **Crash on stray input** and **Documented behaviour mismatch** dismissals assume `elixir_import` supplies `functions`/`macros`/`requires` as sorted ordsets and `elixir_env`/`elixir_expand` supply a map env with `module`/`function`/`file` keys. If `functions`/`macros` were *unsorted*, `ordsets:is_element/2` in `find_import_by_name_arity/2` (line 329) could return a false negative (a real import silently treated as a local), and `find_imports_by_name`'s sorted-order short-circuit (line 305–307) could drop later matches. This is the module's own documented precondition (line 6–7: "assumes it is normalized (ordsets)"), so a violation is a caller-contract break, out of scope here — but it is the one out-of-file invariant this review leans on. Depends on: `elixir_import` (record/import tables stored as sorted ordsets keyed by module) and `elixir_env`/`elixir_expand` (env map key set).

