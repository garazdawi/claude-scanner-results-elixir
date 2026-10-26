# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_fn.erl
<!-- sha256: b505954c645390e3d4cce0e187e3d58e7785f7a15e8c31901e3b040a5941a2f4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T06:17:53Z_

I have enough to write the report. Let me record my analysis of the one notable discrepancy and finalize.

# Bug-hunting review — `elixir_fn.erl`

## 1. Summary

The capture/anonymous-function front-end is defensively written with a proper catch-all on every public dispatch path; the only defect found is that the capture-variable node contradicts its own adjacent comment and suppresses the unused-variable warning it claims to enable.

## 2. Bugfixes

#### `CSCAN-elixir_fn-capture-var-context-suppresses-unused-warning`
- **Finding ID:** CSCAN-elixir_fn-capture-var-context-suppresses-unused-warning
- **Severity:** Low
- **Confidence:** low
- **Location:** `elixir_fn.erl:158`, `escape/3` first clause (the `error ->` branch)
- **Class:** Documented behaviour mismatch (in-file comment vs. code)
- **User-visible outcome:** The comment on lines 150–152 states the capture variable is given a **nil** context "to emit warnings when variable is unused." The code instead builds `Var = {'_&', [{counter, Next}, {capture, Pos} | Meta], ?MODULE}` — the context is `?MODULE` (`elixir_fn`), not `nil`, **and** the variable name `'_&'` begins with an underscore. Both properties independently suppress Elixir's unused-variable diagnostic: a non-nil (module) context marks the var as compiler-generated/hygienic, and an underscore-prefixed name is exempt from unused-variable checks by design. So when a developer writes a capture whose positional argument is genuinely unused (e.g. a capture arg used only as a pattern, the exact case `elixir_env.erl:159` prints a message for), the "variable is unused" warning the comment promises will not fire. The incorrect behaviour a caller would notice: a real unused capture argument compiles with no warning, where the documented intent (and, to the best of my reconstruction, prior behaviour with a `:"&1"`-style nil-context variable) is to warn.
- **Reproduction:** Compile a module containing a capture that binds a positional argument as a pattern it never reads, e.g. `&(&1 = &2)` or a similar shape that leaves `&1` unused after binding; observe that no unused-variable warning is emitted. Note this is a diagnostic-quality regression, not a miscompilation — the resulting function still binds `&1`/`&2` correctly because the same `Var` term (identified by its distinct `counter`) is reused for both the parameter and its references, independent of name/context.
- **Suggested fix:** Restore the variable to the form the comment describes — a `nil` context and a non-underscore, position-distinct name (upstream uses `{list_to_atom([$& | integer_to_list(Pos)]), [{counter, Next} | Meta], nil}`). Minimally, change the context from `?MODULE` to `nil` and drop the leading underscore from the name so unused tracking applies. If the current suppression is intentional, update the comment on lines 150–152 to match, since it currently documents behaviour the code does not implement.
- **Risk of fix:** Re-enabling the warning may surface new "unused variable" warnings in downstream code that previously compiled silently (warnings-as-errors builds could break). Changing the variable name/context also alters the generated variable identity, which could affect macro/quote round-tripping or diagnostic text that inspects the `'_&'` name; verify no other module pattern-matches on the literal `'_&'` atom (grep shows only this site references it).
- **Evidence:**
  - read `elixir_fn.erl:149-160` (comment says "nil context"; code uses `?MODULE` and name `'_&'`).
  - read `elixir_env.erl:159` (unused-variable message specific to capture arguments — confirms captures participate in unused tracking).
  - ran `Grep '_&'` over `src` — only `elixir_fn.erl:158` references the atom, so no other reader depends on the current name/context.

## 3. Risks / caveats

- I could not fetch the canonical upstream `elixir_fn.erl` (no network in this environment), so the claim that the intended form is a `nil`-context `:"&N"` variable is reconstructed from the in-file comment (lines 150–152) plus `elixir_env.erl:159`, not from a diff. The comment-vs-code contradiction is certain; the exact upstream shape and thus the precise magnitude of the regression is inferred — hence Low/low.
- This module is compiler-internal (`elixir_expand` is the only in-folder caller, via `elixir_fn:capture/4`); "documented public API" here means the AST-shaped inputs `capture/4` and `expand/4` accept. All AST-shape dispatch paths in `capture/4` terminate in `invalid_capture/3` (a proper `file_error`), so stray/malformed AST yields a compile error, not a raw crash — no crash-on-input finding.

### Ruled out

**Operator / precedence typo**
- Reason: The only compound boolean/guard expressions (`handle_capture_possible_warning/6` line 191 `andalso` + `/=`; `capture_import/4` line 83 and `capture_require/5` line 100 `andalso`) each read correctly with explicit parenthesisation and match the intended partition (warn iff arity 0 with parens; skip dispatch iff non-sequential).
- Axes examined: no_parens absent, `{no_parens,true}`, `{no_parens,false}`, arity 0 vs >0; ArgsType = arity/sequential/non_sequential | Not examined: none
- Basis: verified

**Missing clause / catch-all**
- Reason: `capture/4` ends with a total catch-all (line 79→`invalid_capture`); `escape/3`, `validate/4`, `check_sequential/2`, `capture_expr/6`, `handle_capture/7`, `fn_arity/1` all have fall-through clauses; `try`/`catch` is not used in this file.
- Axes examined: non-matching AST tuple, integer, atom, list, 2-tuple, `__block__` single/multi, empty args | Not examined: none
- Basis: verified

**Wrong atom / wrong symbol**
- Reason: The default-arg operator atom `'\\\\'` (two backslashes) correctly matches Elixir's `\\` node; error/format_error atoms line up one-to-one between `file_error`/`file_warn` call sites and `format_error/1` heads.
- Axes examined: default-arg detection `is_invalid_arg/1`; every `?MODULE` error tag vs its `format_error` clause | Not examined: none
- Basis: verified

**State / accumulator threading error**
- Reason: `expand/4` threads `S`→`SReset`→`SAcc`→merged via `mapfoldl`, then reads `SE` for the version counter and returns `SE#elixir_ex{version=Counter+1}`; `capture_require/5` threads `SE`/`EE` from `elixir_expand:expand` into `handle_capture`; the `orddict` accumulator in `escape/3` is threaded through `mapfoldl` and both 3-tuple/2-tuple recursive clauses (`Dict0`→`Dict1`→`Dict2`).
- Axes examined: single vs multi clause fn; capture with 1 vs 2 positional args; nested `+` re-ordered args (`&(&2+&1)`) | Not examined: none
- Basis: verified

**Off-by-one / wrong arithmetic**
- Reason: `fn_arity/1` subtracts exactly the guard element (`length(Args)-1` for `when`); `validate/4` starts at position 1 and increments by 1; `args_from_arity/3` uses `lists:seq(1,A)` giving `[]` at A=0; `check_sequential/2` increments correctly; arity bound `0..255` matches Elixir's max arity.
- Axes examined: arity 0, 1, 255, 256; guarded vs unguarded clause; `&1`-only vs `&1,&2`; skipped `&2` | Not examined: none
- Basis: verified

**Contract violation**
- Reason: All three return shapes (`{expand,Fn,S,E}`, `{{remote,...},Meta,DotMeta,S,E}`, `{{local,...},...}`) are exactly the shapes `elixir_expand:expand_fn_capture/4` (lines 651–660) pattern-matches; `expand/4` returns the documented `{Expr,S,E}` triple.
- Axes examined: remote-arity, local-arity, sequential-import, non-sequential expr, complex-module fallback | Not examined: none
- Basis: verified

**Crash on stray / unexpected input**
- Reason: Every `capture/4` dispatch path reaches either a guarded specific clause or the line-79 catch-all producing `file_error`; `capture_require/5`'s single head is only reached from callers that already guarantee the `{{'.',_,[L,R]},_,_}` shape (lines 49, 56).
- Axes examined: integer capture arg, bare atom, 2-tuple, list, `.`/2 dot, anonymous `.`/1 dot, `__block__` | Not examined: `capture_require/5` invoked with a non-dot term (only reachable by an internal caller violating the shape — out of scope per instructions)
- Basis: verified

**Resource leak on error path**
- Reason: No files, ETS tables, processes, monitors, or counters are opened/held here; `next_counter/1` only increments an existing counter. No cleanup path exists to leak.
- Axes examined: error branch of `escape`, `args_from_arity`, `validate` | Not examined: none
- Basis: verified

**Silent corruption**
- Reason: See the reported finding — the one silent-behaviour deviation (variable context/name) is reported; it does not corrupt binding (same `Var` term reused for parameter and references), only suppresses a warning.
- Axes examined: variable identity across parameter/body reuse; distinct counters per position | Not examined: none
- Basis: verified

**Infinite loop / hang**
- Reason: All recursion (`escape/3`, `validate/4`, `check_sequential/2`, `capture/4`→`capture/4` rewrite on `__block__`/2-tuple) strictly reduces the term or list length; no `receive`.
- Axes examined: nested dot, nested tuple rewrite, list of args | Not examined: none
- Basis: verified

**Quadratic-where-linear**
- Reason: No `++` in a loop, no per-iteration `length` in a hot path (`length/1` calls are one-shot per clause on ≤255-element arg lists), no pattern recompilation.
- Axes examined: `escape` mapfoldl, `validate` cons-build (bounded ≤255) | Not examined: none
- Basis: verified

**Repeated work**
- Reason: No compiled patterns, BEAM reads, or cache recomputation in this module.
- Axes examined: whole file | Not examined: none
- Basis: verified

**Dead / unreachable code**
- Reason: All clauses are reachable from `elixir_expand:expand_fn_capture/4`; the two `%% TODO` clauses (complex-module capture, line 128–132; complex-module warning, line 96–97) are live compatibility paths, not dead code.
- Axes examined: each `capture/4` head, `capture_expr/6` clauses | Not examined: none
- Basis: verified

**Documented behaviour mismatch**
- Reason: Reported as the sole finding (line 158 vs comment lines 150–152).
- Axes examined: variable context nil vs `?MODULE`; name underscore-prefixed vs not | Not examined: none
- Basis: verified

**Race condition / shared mutable state**
- Reason: The only shared state touched is `next_counter/1` (atomic `ets:update_counter` or `unique_integer`), which is inherently concurrency-safe; no read-then-write.
- Axes examined: concurrent compiles calling next_counter | Not examined: none
- Basis: verified

**Wrong scope / order of operations**
- Reason: `handle_capture_possible_warning` is invoked before `capture_require` (line 48 before 49) as intended; `escape`-then-expand ordering in `capture_require/5` is required (module must be escaped before expansion); no side effect precedes its input validation.
- Axes examined: warning-before-dispatch, escape-before-expand ordering | Not examined: none
- Basis: verified

**Encoding / unicode mishandling**
- Reason: This module manipulates AST tuples only; text handling is delegated to `'Elixir.Macro':to_string/1` in `format_error/1`. No byte/codepoint logic here.
- Axes examined: format_error string building | Not examined: none
- Basis: verified

**Unclassified**
- Reason: No composite or file-specific defect beyond the reported comment/behaviour mismatch was found.
- Axes examined: cross-clause interactions in `capture`/`capture_require`/`capture_expr` | Not examined: none
- Basis: verified

### Did I verify this?

- **Operator/precedence, Missing clause, Wrong atom, State threading, Off-by-one, Contract violation, Crash-on-input, Resource leak, Silent corruption, Infinite loop, Quadratic, Repeated work, Dead code, Documented-behaviour, Race, Wrong-scope, Encoding, Unclassified** — all marked `verified`: I read every clause of `elixir_fn.erl` (lines 1–233), traced triggers from the `elixir_expand:expand_fn_capture/4` entry point (`elixir_expand.erl:650–661`), confirmed the return-shape contract against that caller, confirmed `next_counter/1` semantics (`elixir_module.erl:64–71`), and confirmed the capture-var atom has no other reader via grep. Each entry's `Not examined` reads `none`, so no `Depends on:` lines remain.
- One residual external dependency, not a ruling-out: the finding's severity rests on the intended upstream variable form, which I inferred from the in-file comment and `elixir_env.erl:159` rather than a fetched upstream diff — this is disclosed in the finding (Confidence: low) and in Risks/caveats, and does not affect any `verified` dismissal above (the comment-vs-code contradiction itself is directly observed in-file).

