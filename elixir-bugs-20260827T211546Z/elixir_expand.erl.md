# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_expand.erl
<!-- sha256: 1fcb8641377ffab1950db04e717abd15d11c0abd380f920ba4a72ee2d167011e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T06:17:00Z_

Based on a full read of both halves of the file (lines 1–1096 and 1097–1366), here is my review.

# Bug-hunting review — `elixir_expand.erl`

## Summary
A full read of the macro-expander's public entry points (`expand/3`, `expand_args/3`, `expand_arg/3`, `format_error/1`) and every helper they reach surfaced no defect that makes the module crash, corrupt state, hang, or violate its contract on inputs its callers legitimately produce.

## Bugfixes

No findings.

The most defect-prone areas for this file — state/accumulator threading (`S`/`SR`/`SL`/`ST`/`SU` and `E`/`ER`/`ET`/`EU`/`EQ` handoffs), version-counter arithmetic, guard connectives, and error-tag/`format_error` coverage — were each traced and found consistent:

- **State threading** in `=` (l.16–20), `alias`/`require`/`import` (l.72–187), remote/local dispatch (l.525–551, 967–1006), and comprehensions (l.854–925) each thread the *post*-expansion state/env forward and return the correct halves. `require` deliberately returns the pre-require scope `ST` (require doesn't mutate scope) while advancing the env `EU`; `for` deliberately returns the *original* `E` because comprehension bindings are scoped (l.879). Both match the intended semantics rather than being dropped accumulators.
- **Version counters** for `cond`/`receive`/`try`/`case`/`for` (l.344, 354, 360, 820, 877–879) all bind `Counter` from the returned scope and emit `Counter+1` — uniform, no off-by-one.
- **Guard connectives** — the one OR-guard that matters, `validate_for_options([], Into, Uniq, {reduce,_}, …) when Into /= false; Uniq /= false` (l.889), correctly flags reduce-conflicts-with-into-or-uniq; clause ordering (l.881–899) leaves the plain reduce case to the `{ok, Reduce, …}` catch-all.
- **`format_error/1`** (l.1169–1365) covers every tag raised in this file; every `file_error`/`module_error`/`function_error` reason has a matching clause.
- **Arity discipline** — `elixir_env:prepare_write/1` (l.568, 744) vs `prepare_write/2` (l.530, 540) is intentional (list/args vs remote-call paths), not a typo; `file_error/4` is called with both an env map (l.38 …) and a bare file binary (l.586, 600, 1129), matching how `elixir_errors:file_error` is used across the folder.

## Risks / caveats
- No `-spec` declarations exist in this file, so the spec-totality obligation is vacuous; correctness rests on the callers in the call-graph slice (`elixir`, `elixir_clauses`, `elixir_dispatch`, `elixir_fn`, `elixir_map`, …) delivering the documented AST shapes. I did not execute Erlang (no confirmed toolchain), so all conclusions are read-and-trace, not runtime-verified.
- Several dismissals below depend on neighbour-module contracts (`elixir_clauses:match/6`, `elixir_env:prepare_write`/`close_write`/`reset_read`, `elixir_errors:file_error/4` accepting either env or file). Those are named under `Depends on:` for the cross-file pass.

### Ruled out

**Operator / precedence typo**
Reason: The only semantically load-bearing `;`/`,` guard (`validate_for_options`, l.889) and the boolean short-circuits (l.85, 160, 269, 580–582, 985) partition inputs as intended.
Axes examined: reduce+into set, reduce+uniq set, reduce alone, into=false/return=true, uniq=true/return=false | Not examined: none
Basis: verified

**Missing clause / catch-all**
Reason: `expand/3` terminates in total catch-alls (l.555 invalid call, l.599 invalid quoted expr); `quote` (l.222/230/233/304), `for` options, and var clauses each have fall-throughs.
Axes examined: 0-arg/1-arg/2-arg quote, non-list opts, empty block `__block__ []`, empty args anonymous call, tuple/list/number/atom/binary/pid/fun literals | Not examined: none
Basis: verified

**Wrong atom / wrong symbol**
Reason: Meta keys (`ambiguous_op`, `if_undefined`, `defined`, `location`, `bind_quoted`, `generated`, `no_parens`, `context`, `runtime_module`, `optimize_boolean`, `keep_unused`) and env keys (`module`, `file`, `function`, `context`, `lexical_tracker`, `context_modules`) are used consistently with how they're read/written.
Axes examined: each keyfind/keymember/`?key` target cross-checked against its writer in-file | Not examined: cross-module key producers (elixir_env/elixir_aliases)
Basis: asserted
Depends on: `elixir_env.erl`/`elixir_aliases.erl` — env map keys (`context_modules`, `lexical_tracker`, `runtime_modules` in `#elixir_ex`) match the names read here.

**State / accumulator threading error**
Reason: Every multi-step clause forwards the latest `S*`/`E*` and returns the pair the construct's scoping semantics require (see Bugfixes narrative).
Axes examined: `=`, alias/require/import (all sub-branches), remote nil/guard/erlang-const paths, for reduce/non-reduce, quote prelude/context/binding | Not examined: none
Basis: verified

**Off-by-one / wrong arithmetic**
Reason: All version bumps are `Counter+1` bound from the returned scope; `length(Args)` arities feed `resolve_super`/traces consistently.
Axes examined: cond/receive/try/case/for counters, super arity match | Not examined: none
Basis: verified

**Contract violation**
Reason: `expand/3` always returns a `{Expr, S, E}` triple or raises via `file_error`; `format_error/1` returns iodata for all tags.
Axes examined: every return arm of `expand/3` and each `format_error` clause | Not examined: none
Basis: verified

**Crash on stray / unexpected input**
Reason: Hostile/odd AST is funnelled to `{invalid_call,…}` (l.555) or `{invalid_quoted_expr,…}` (l.599); `is_list(Args)`/`is_atom` guards gate the structured clauses.
Axes examined: 4-tuple, foreign map, float 0.0 in/out of match, pid in/out of function, external vs closure fun | Not examined: deeply malformed nested metadata lists
Basis: asserted
Depends on: the yecc parser (`elixir_parser`) and quote machinery producing 3-element `{Form, Meta, Args}` nodes with list `Meta`; raw user terms reach `expand/3` only via `Macro.escape`-shaped data.

**Resource leak on error path** / **Infinite loop / hang** / **Race condition / shared mutable state**
Reason: Pure transformation module — opens no files/ETS/processes/monitors, spawns nothing, all recursion structurally decreases over AST/option lists, and holds no shared mutable state (config reads at l.496 are single lookups).
Axes examined: every recursive helper (`expand_list`, `expand_block`, `mapfold`, `validate_for_options`) decreases its list arg | Not examined: none
Basis: verified

**Silent corruption**
Reason: `is_useless_building`, `escape_map` (sorts), `escape_env_entries`, and the `quote` binding builder produce the documented shapes; no accumulator is dropped on an empty-collection edge (`__block__ []`→`nil`, empty binding→`EQuoted`).
Axes examined: empty binding list, empty prelude, empty block, single-element block | Not examined: none
Basis: verified

**Quadratic-where-linear / Repeated work**
Reason: Accumulators use `[H|Acc]` + `lists:reverse` (`expand_list`, `expand_block`, `mapfold`); no `++` in a per-element loop (the `++` at l.878/1111 are one-shot list joins), no pattern recompilation.
Axes examined: each recursive accumulator | Not examined: none
Basis: verified

**Dead / unreachable code**
Reason: `format_error({invalid_clauses,…})` (l.1292) and `{nested_comparison,…}` (l.1339) have no raiser in this file but are shared error formatters reached from sibling modules; not dead in the program.
Axes examined: grepped each format_error tag against in-file raisers | Not examined: cross-module raisers (assumed present)
Basis: asserted
Depends on: `elixir_clauses`/`elixir_dispatch` raising `invalid_clauses`/`nested_comparison` through `?MODULE` format_error.

**Documented behaviour mismatch**
Reason: The module carries no `-spec`/`@doc` prose contract to contradict; behaviour is defined by the language semantics the code implements.
Axes examined: n/a (no documented claims in-file) | Not examined: n/a
Basis: verified

**Wrong scope / order of operations**
Reason: `=` expands RHS before matching LHS (l.18–19, Elixir semantics); pin resolution swaps read-vars to the prematch set before expanding the pinned expr (l.386); side-effecting traces occur after scope assertions.
Axes examined: `=`, `^`, super, quote unquote-in-pattern guard ordering | Not examined: none
Basis: verified

**Encoding / unicode mishandling**
Reason: No byte/codepoint handling here; strings pass through as opaque binaries/atoms (identifier/Unicode enforcement lives in `elixir_tokenizer`, out of this file's scope).
Axes examined: binary literal passthrough (l.577) | Not examined: none (not applicable)
Basis: verified

**Unclassified**
Reason: No file-specific one-off sin surfaced after tracing the quote binding construction, `__ENV__` field access (l.206–213), and rewrite-case-clause optimizer (l.822–850).
Axes examined: `__ENV__.field` present/absent key, optimize_boolean on/off, bind_quoted valid/invalid | Not examined: none
Basis: verified

### Did I verify this?

- **Operator/precedence, Missing clause, State threading, Off-by-one, Contract, Leak/hang/race, Silent corruption, Quadratic, Documented-behaviour, Wrong-scope, Encoding, Unclassified** — kept `verified`: I read the relevant clauses and tried the enumerated axes to exhaustion (`Not examined: none`), reasoning through each input shape by hand.
- **Wrong atom/symbol** — kept `asserted`. A read confirms in-file producer/consumer parity, but the untried axis (env/meta keys minted by `elixir_env`/`elixir_aliases`) is genuinely out-of-file; verification belongs in the cross-file pass against those modules' reports. `Depends on:` names them.
- **Crash on stray input** — kept `asserted`. The untried axis is deeply malformed nested metadata, which only reaches `expand/3` if the parser/quote layer emits non-conforming nodes; that guarantee lives in `elixir_parser`/`elixir_quote`, named under `Depends on:`.
- **Dead code** — kept `asserted`. Whether the two orphan `format_error` tags are truly reached requires reading `elixir_clauses`/`elixir_dispatch`; named under `Depends on:`. Re-reading this file alone cannot upgrade it.

