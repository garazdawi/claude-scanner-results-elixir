# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_import.erl
<!-- sha256: fd10160271dafb47711fb0f4c5e03ffe5ed8d4d86c81aa867cd036edee1d6292 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_import.erl
<!-- sha256: fd10160271dafb47711fb0f4c5e03ffe5ed8d4d86c81aa867cd036edee1d6292 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T (Erlang/OTP + Elixir 1.21.0-dev toolchain available and used)_

# Bug-hunting review: `elixir_import.erl`

## Summary
Two defects confirmed — both now **reproduced end-to-end** against a live Elixir 1.21.0-dev compiler, upgrading them from medium to high confidence: (1) the `only: :sigils` path crashes the compile on any target module lacking `__info__/1` (e.g. every Erlang module), where the `:functions`/`:macros` paths recover; (2) an explicitly-listed import that collides with a special form emits a second, factually-wrong "undefined or private" warning. The re-review resolved all six previously-asserted ruled-out entries: four were upgraded to `verified` with executed evidence, two remain `asserted` because their untried axis is genuinely out of contract / not executable.

## Bugfixes

#### `CSCAN-elixir_import-sigil-path-missing-undef-fallback`
- **Finding ID:** `CSCAN-elixir_import-sigil-path-missing-undef-fallback`
- **Severity:** Low
- **Confidence:** high _(upgraded from medium — reproduced end-to-end)_
- **Location:** `elixir_import.erl:109-117` (`import_sigil_functions/6`, `import_sigil_macros/6`), contrast `:171-183` (`get_functions/2`, `get_macros/1`)
- **Class:** Crash on stray / unexpected input (missing catch-all)
- **User-visible outcome:** For `import Foo, only: :functions` / `only: :macros`, a target whose `__info__/1` is unavailable is handled gracefully — `get_functions/2` catches `error:undef` and falls back to `module_info(exports)`, `get_macros/1` catches it and returns `[]`. The sigil path calls `InfoCallback(functions)` / `InfoCallback(macros)` directly inside `filter_sigils/1` with no `try`, so the same input raises an uncaught `undef` and aborts the compile with an internal error instead of importing zero sigils.
- **Reproduction (executed):** `elixir -e 'defmodule B do import :lists, only: :sigils end'` raises:
  ```
  ** (UndefinedFunctionError) function :lists.__info__/1 is undefined or private
      (elixir) src/elixir_import.erl:111: anonymous fn/1 in :elixir_import.import_sigil_functions/6
      (elixir) src/elixir_import.erl:120: :elixir_import.calculate_except/7
      (elixir) src/elixir_import.erl:61:  :elixir_import.import_only_except/7
  ```
  The matching `import :lists, only: :functions` compiles cleanly (only an "unused import" warning), proving the asymmetry. Any Erlang module (which has `module_info` but no `__info__/1`) triggers it — trivially reachable, no exotic input required.
- **Suggested fix:** Route the sigil thunks through the guarded accessors, e.g. `filter_sigils(get_functions(Ref, InfoCallback))` and `filter_sigils(get_macros(InfoCallback))`, so the `undef` fallback applies uniformly.
- **Risk of fix:** none — for a module that answers `__info__/1` the results are identical; it only changes the no-`__info__` case from crash to empty import, matching the sibling paths.
- **Evidence:** read `elixir_import.erl:109-117,171-183`; reproduced the crash and the non-crashing sibling path against Elixir 1.21.0-dev; the stack trace pins the fault at `:111`, exactly the sigil thunk.

#### `CSCAN-elixir_import-special-form-double-warning`
- **Finding ID:** `CSCAN-elixir_import-special-form-double-warning`
- **Severity:** Low
- **Confidence:** high _(upgraded from medium — reproduced end-to-end)_
- **Location:** `elixir_import.erl:69-73` (`import_only_except/7`, `{only, DupOnly}` clause) together with `:215-224` (`ensure_no_special_form_conflict/5`)
- **Class:** Documented behaviour mismatch (incorrect diagnostic text)
- **User-visible outcome:** When a name/arity listed in `only:` is genuinely exported by the target but is also an Elixir special form, `ensure_no_special_form_conflict/5` removes it and emits the correct `special_form_conflict` warning. Because the discarded pair is no longer in `Used1`/`Used2`, the residue `(Only -- Used1) -- Used2` at `:71-72` still contains it, so a *second* `invalid_import` warning fires claiming the function is "undefined or private" — false, since it is defined and exported.
- **Reproduction (executed):** Erlang module `sfmod` exporting `with/1` (`special_form('with', _) -> true`), then `import :sfmod, only: [with: 1]` produces both warnings:
  ```
  warning: cannot import :sfmod.with/1 because it conflicts with Elixir special forms, the import has been discarded
  warning: cannot import :sfmod.with/1 because it is undefined or private
  ```
  The second is factually wrong for an exported function.
- **Suggested fix:** Before computing the residue, subtract the pairs dropped as special-form conflicts (surface the discarded set from `calculate_key`/`ensure_no_special_form_conflict`, or filter `Only` through `special_form/2` when building the unused list).
- **Risk of fix:** Low — removes one incorrect warning line for an uncommon input; any test asserting the current double-warning would need updating.
- **Evidence:** read `elixir_import.erl:69-73,140,215-224,246-248`; reproduced both warnings against Elixir 1.21.0-dev.

## Risks / caveats
- **No `-spec` declarations exist in this file**, so the spec-totality checklist reduces to the exported arities: `import/6`, `import/7`, `special_form/2`, `record/4`, `ensure_no_local_conflict/3`, `format_error/1`. Each was traced and covers its input space (see Ruled out).
- **Reachability of both findings is now confirmed, not assumed.** The sole in-folder caller of the import path is `elixir_expand.erl:157`, which calls `elixir_import:import/6`; `import/6` (`:16`) always builds `fun Ref:'__info__'/1` as the `InfoCallback`. `elixir_expand` accepts any compile-time atom module (`:152`), including Erlang modules with no `__info__/1`, so finding #1's crash and finding #2's exported-special-form collision are both reachable through the normal compiler front-end — as the end-to-end reproductions demonstrate.

### Ruled out

**Operator / precedence typo**
- Reason: The only compound guards (`record/4:146-147` using `;`, `is_sigil/1:192-197` mixing `,`/`andalso`/`orelse`) partition inputs as intended.
- Axes examined: `Function == nil`, `Module == Receiver`, single lowercase sigil letter, single uppercase letter (empty tail → `lists:all(_,[])`=true), multi-char upper+digit tail, non-`sigil_` name | Not examined: none
- Basis: verified

**Missing clause / catch-all**
- Reason: `import_only_except/6` covers `false`/`{except,list}`/`{except,Other}`; `import_only_except/7` covers all `only` atoms, list, `{only,Other}`, and `false`; `special_form/2` ends in `(_,_)->false`.
- Axes examined: `only` = functions/macros/sigils/list/other-atom/absent, `except` = list/non-list/absent, both-given combinations | Not examined: none
- Basis: verified

- **Wrong atom / wrong symbol** — _Upgraded from hedged assumption in first pass (`asserted` → `verified`)_
- Reason: `special_form/2` entries and arities cover the complete current Elixir special-form set; record/env keys (`functions`, `macros`, `{import, Tuple}`) are used consistently across read and write sites.
- Axes examined: each special-form atom/arity pair; `?key(E,functions)`/`?key(E,macros)` producer vs consumer sites; **exhaustive cross-check of `special_form/2` against the canonical `Kernel.SpecialForms` set** | Not examined: none
- Basis: verified
- Executed evidence: compiled the actual `elixir_import.erl` (`erlc -I …/src`), enumerated all 31 entries of `Kernel.SpecialForms.__info__(:macros)` from Elixir 1.21.0-dev (same version as the tree), and applied `elixir_import:special_form/2` to each — **0 missing** (`MISSING from special_form/2: []`). The file is additionally a superset (`|`/`->` are special forms not surfaced by `__info__`), so no special form can slip through unguarded. The prior `Depends on` (external canonical list) is now discharged by running both.

**State / accumulator threading error**
- Reason: `E`/`EI` threading in `import/7` is single-pass; `Added1 or Added2`, `Used1`/`Used2`, and the `{Bool,Set,List}` triples from `calculate_key/6` are destructured to the correct positions.
- Axes examined: functions-only, macros-only, sigils (both), list-only, plain import return tuples | Not examined: none
- Basis: verified

**Off-by-one / wrong arithmetic**
- Reason: No index/length/offset arithmetic in the file; only set operations and list membership.
- Axes examined: whole-file scan for arithmetic operators | Not examined: none
- Basis: verified

**Contract violation**
- Reason: `import/7` returns `{ok, Added, RequireResult}` or `{error, Reason}` on every path; `record/4` returns boolean; `ensure_no_local_conflict/3` returns a list; `format_error/1` returns iodata.
- Axes examined: each return site of every exported function | Not examined: none
- Basis: verified

**Crash on stray / unexpected input**
- Reason: Reported as finding #1 (sigil path `undef`, now reproduced). Other input paths are guarded (`catch error:badarg`, `catch error:undef`, `ensure_keyword_list/1` validating shape before use).
- Axes examined: non-keyword `only`/`except` values, non-list, module without `__info__` for functions/macros/sigils (all three exercised live) | Not examined: `InfoCallback` returning a non-list term
- Basis: asserted
- Depends on: The sole caller chain (`elixir_expand.erl:157` → `elixir_import:import/6` → `import/7` with `fun Ref:'__info__'/1`) means `InfoCallback` is *always* a module's `__info__/1`, never a caller-supplied custom fun; a non-list return would require a module whose `__info__(functions)` returns a non-list, which no real module produces. Reading the caller confirms the callback is fixed but does **not** push a non-list value through `intersection/2`/`filter_sigils/1`, so the axis stays untried and the entry stays `asserted` (out-of-contract, caller-guaranteed).

**Resource leak on error path**
- Reason: The module creates no processes/handles/monitors; ETS tables are owned by `elixir_module`, only inserted into / looked up here.
- Axes examined: `record/4` insert path, `ensure_no_local_conflict/3` lookup path | Not examined: none
- Basis: verified

**Silent corruption**
- Reason: Set math (`ordsets:from_list`, `--`, `intersection/2`, `keydelete/2`) preserves the intended membership; empty-set path returns `{false,[],...}` correctly dropping the key.
- Axes examined: empty `New`, non-empty `New`, pre-existing key in `Old`, absent key | Not examined: none
- Basis: verified

- **Infinite loop / hang** — _Upgraded from hedged assumption in first pass (`asserted` → `verified`)_
- Reason: All recursion (`ensure_keyword_list/1`, `intersection/2`) strictly consumes list heads; no `receive`, no unbounded retry.
- Axes examined: empty list, single element, long list, **improper-tail input to `intersection/2`** | Not examined: none
- Basis: verified
- Executed evidence: ran `intersection([{a,1}|{b,2}], [{a,1}])` (the improper-tail axis) → it raises `function_clause` immediately (crash), **not** a hang. So even outside the `ensure_keyword_list/1` gate the worst case is a bounded crash, never an infinite loop. Prior `Depends on` (the upstream gate) is no longer needed to rule out a hang.

- **Quadratic-where-linear** — _Upgraded from hedged assumption in first pass (`asserted` → `verified`) for the named axis_
- Reason: `intersection/2` is `O(|Only| · |All|)`; `Only` is a source-literal keyword list (bounded by source size), `All` is a module's export list.
- Axes examined: `intersection/2` build shape, `ensure_no_duplicates/5` `lists:member` visited-set, **pathological module with tens of thousands of exports** | Not examined: none
- Basis: verified
- Executed evidence: ran `intersection(Only, All)` with `|All| = 50 000` and `|Only| = 2` → completed in **0 ms**. Because `Only` is a literal keyword list written in source, it cannot independently reach a size that makes the product super-linear (a 50 000-entry `only:` list is not an expressible input at scale); the named pathological export-count axis is fast. Prior `Depends on` (caller list sizes) discharged.

**Repeated work**
- Reason: No compiled patterns/regex; `get_functions`/`get_macros` call `__info__` once per import.
- Axes examined: per-call recomputation sites | Not examined: none
- Basis: verified

- **Dead / unreachable code** — _Upgraded from hedged assumption in first pass (`asserted` → `verified`)_
- Reason: All clauses are reachable from `import/6,7` for corresponding `only`/`except` shapes; the `format_error({no_macros,_})` clause at `:263` has no producer anywhere in the codebase — a harmless orphan formatter clause, not a bug.
- Axes examined: each `import_only_except` branch, each `format_error` reason, **whether `{no_macros,_}` is produced anywhere in the codebase** | Not examined: none
- Basis: verified
- Executed evidence: repo-wide grep for `no_macros` across the entire `/Users/lukas/code/claude-scanner/elixir` tree returns exactly one hit — the `format_error/1` clause itself (`elixir_import.erl:263`). No module raises/emits `{no_macros, _}`, so the clause is confirmed dead. A never-invoked `format_error` clause returns iodata harmlessly if ever reached; it is dead-but-inert, not a defect. Prior `Depends on` (some producer exists) is disproven, and the conclusion (not a bug) stands.

**Documented behaviour mismatch**
- Reason: The `only_and_except_given` message matches the code, which applies `except` in the three atom branches and rejects `only`-list + `except`. The one prose-vs-code gap (special-form entries producing a false "undefined or private" warning) is reported as finding #2.
- Axes examined: `only` atom + `except` combination, `only` list + `except` rejection, `invalid_option` messages vs producing clauses | Not examined: none
- Basis: verified

**Race condition / shared mutable state**
- Reason: ETS writes/reads go through tables owned by `elixir_module`; this module does no read-then-write and holds no process state.
- Axes examined: `record/4` insert, `ensure_no_local_conflict/3` lookup_element | Not examined: concurrent compile of the same module inserting the same `{import,Tuple}` key
- Basis: asserted
- Depends on: `elixir_module:data_tables/1` (`elixir_module.erl:34-35`) returns per-module tables keyed by `Module` in the `elixir_modules` table, so concurrent `record/4` inserts of the same `{import,Tuple}` resolve to the same `Receiver` value (value-stable last-writer-wins). Reading `elixir_module` confirms the per-module keying but does **not** execute the concurrent-insert interleaving, which is not reproducible with the available single-node tooling; the axis stays untried and the entry stays `asserted`.

**Wrong scope / order of operations**
- Reason: The one ordering wrinkle — `ensure_no_duplicates/5` for `except` runs before `import_only_except/7` may return `only_and_except_given`/`invalid_option` — produces at most a redundant warning ahead of an error, not incorrect state.
- Axes examined: except-dedup-then-only-error path, except-dedup-then-success path | Not examined: none
- Basis: verified

**Encoding / unicode mishandling**
- Reason: Names are atoms; `atom_to_list/1` in `is_sigil/1`/`remove_underscored/1` inspects ASCII prefixes (`"sigil_"`, `"_"`) and ASCII letter/digit ranges only, correct regardless of any Unicode tail bytes.
- Axes examined: `"sigil_" ++ Letters` with ASCII tail, non-ASCII atom name (prefix check still valid), `"_"`-prefixed name | Not examined: none
- Basis: verified

### Did I verify this?

- **Wrong atom / wrong symbol** — upgraded `asserted` → **verified**. Executed: compiled the real file and ran `elixir_import:special_form/2` against all 31 `Kernel.SpecialForms.__info__(:macros)` entries from the matching Elixir version; 0 missing. Both implementations run and compared.
- **Infinite loop / hang** — upgraded `asserted` → **verified**. Executed the improper-tail axis into `intersection/2`; result is a `function_clause` crash, not a hang.
- **Quadratic-where-linear** — upgraded `asserted` → **verified** for the named axis. Executed `intersection/2` at `|All|=50 000`; 0 ms. `Only` is a source-literal list, bounding the product.
- **Dead / unreachable code** — upgraded `asserted` → **verified**. Repo-wide grep shows no producer of `{no_macros,_}`; the formatter clause is confirmed dead-but-inert.
- **Crash on stray input** — kept `asserted`. Confirmed by reading `elixir_expand.erl:157`/`elixir_import.erl:16` that `InfoCallback` is always `Ref:'__info__'/1`, but the non-list-return axis is out of contract and was not pushed; reading the caller does not clear the axis.
- **Race condition** — kept `asserted`. Confirmed per-module table keying via `elixir_module.erl:34-35`, but the concurrent-insert interleaving is not executable here; the axis stays untried.
- **All other entries** — kept `verified`; each `Not examined` reads `none` and the cited clauses were read this pass.

### Dropped in re-review
None. Both prior findings are preserved and strengthened (medium → high) by end-to-end reproductions; no ruled-out entry was converted into a finding, because the two entries with untried axes (crash-on-stray `InfoCallback` non-list, race concurrent-insert) rest on inputs that are respectively out of contract (caller-fixed to `__info__/1`) and not executable with available tooling — insufficient evidence to promote, so they remain low-noise dismissals for a later pass.
