# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_map.erl
<!-- sha256: baca864a16e9f062bd08aff249c5c68f5af3db81509b04d8c3e430f42bb06c42 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_map.erl
<!-- sha256: baca864a16e9f062bd08aff249c5c68f5af3db81509b04d8c3e430f42bb06c42 -->

# Security Review: `elixir_map.erl`

_Re-analysed on 2026-08-28 (hedge-resolution pass, second iteration)_

## Summary

`elixir_map.erl` is the compiler's map/struct expansion pass, reachable **only** through `elixir_expand` during compilation/eval. It contains no deserialization/atom-creation/file/command/network sinks and turns every malformed AST run through it into clean compile diagnostics. The core verdict is unchanged: **no high- or medium-confidence security defect.**

This pass re-checked the single surviving hedge — the blocking wait behind `wait_for_struct` on `Elixir.Kernel.ErrorHandler:ensure_compiled/4`. I re-ran the neighbour reads and repo search:
- `elixir_env.erl:33-34` (`trace/2`) was re-read and confirms the trace-routing claim (kept `verified`).
- `Elixir.Kernel.ErrorHandler` and `ensure_compiled` return **no files** across the entire `/Users/lukas/code/claude-scanner/elixir` tree (`grep -rl` over `.ex`/`.erl`, plus `find -iname '*error_handler*'` → empty). The module is compiled elsewhere; its cycle-breaking logic is not present in this checkout and cannot be read or executed here.

Because the deadlock-avoidance guarantee that path relies on is **out-of-file, out-of-checkout, and not a callgraph neighbour**, it remains an unverifiable cross-file assumption and stays recorded as a **low-confidence finding** rather than a silent ruled-out — the safe direction per the re-review rule. Everything else is `verified` with `Not examined: none` (axes actually executed in the prior pass; nothing in the file changed, sha256 identical).

## Security

### Finding (low confidence) — Blocking wait on `Elixir.Kernel.ErrorHandler:ensure_compiled/4` (`wait_for_struct`, `elixir_map.erl:242-244`) — *Upgraded from hedged assumption in first pass*

`wait_for_struct/4` gates struct expansion on
`'Elixir.Kernel.ErrorHandler':ensure_compiled(Module, struct, Mode, Line) =:= found`, which blocks the compiling process while another module compiles. The dismissal in the original pass rested on the assumption that the error handler breaks compilation cycles and returns a terminal (non-`found`) status rather than blocking indefinitely — surfacing `inaccessible_struct`/`undefined_struct` instead of deadlocking.

Verification attempted this pass:
- `Elixir.Kernel.ErrorHandler` is **not** in this file's callgraph neighbour set (`elixir_aliases`, `elixir_def`, `elixir_env`, `elixir_expand`, `elixir_module`, `elixir_quote`, `elixir_utils`), so the Read-the-neighbours route does not reach it.
- `grep -rl "ensure_compiled"` over `.ex`/`.erl`, `grep -rln "Kernel.ErrorHandler"`, and `find -iname '*error_handler*'` across the checkout **all returned empty**. The module's source is not in this tree, so its cycle-breaking logic can neither be read nor executed from here.

I therefore cannot confirm that a maliciously-constructed cyclic `use`/struct-reference graph in attacker source cannot drive this into an indefinite block (a compile-time DoS of the compiling process). Circumstantial evidence still points to a terminal return: the `Mode = hard/soft` argument, and the `format_error/1` text at `elixir_map.erl:296` ("you likely have cyclic module usage in your code") imply the handler returns and the caller reports rather than hangs. The threat model also already grants the source author compile-time code execution, so the uplift is small. But an unverifiable out-of-checkout guarantee must be a low-confidence finding, not an assertion.
- **Sink class:** Resource / DoS (unbounded blocking wait).
- **Confidence:** low. The defect is speculative and depends entirely on `ensure_compiled/4`'s cycle handling, which is out-of-file, absent from this checkout, and trusted-compiler-internal. A cross-file pass with access to `Kernel.ErrorHandler` (the `.ex` source) should confirm `ensure_compiled/4` returns a terminal status on a cycle instead of blocking, at which point this downgrades back to a clean ruled-out.

Aside from this one item, **no findings**: every other security-relevant operation (dynamic `Name:'__struct__'/1` dispatch, `code:ensure_loaded/1`, the `unknown_key`/`invalid_struct_return_value` validations, the AST-walking recursions) is reached only after `elixir_expand` has begun expanding a `%{}`/`%` node during `Code.compile_*`/`Code.eval_*`/`Macro.expand*`, where the source supplier already holds compile-time code execution. None of the malformed inputs run in the prior pass produced an internal crash, atom-table growth, unbounded resource use, or a validation bypass. Details per sink are in **Ruled out**.

## Risks / caveats

- **Threat-model dependency.** The verdict rests on the folder threat model: macro expansion is not a sandbox, and reaching `elixir_map` requires compiling/evaluating the attacker's source (`elixir_expand.erl` is the only in-folder caller). A downstream tool that ran `elixir_map:expand_struct/5` on untrusted AST *without* compilation's code-execution semantics would turn the compile-time `Name:'__struct__'(Assocs)` / `code:ensure_loaded(Name)` dispatch (`elixir_map.erl:155,185,197,199,235`) into an untrusted-input code-execution/module-load surface. No such caller exists in-tree.
- **Diagnostic-quality nuance (not a security issue).** For `defstruct`-generated structs, the strict auto-generated `__struct__/1` (`Map.replace!`) raises `KeyError` on an unknown key *before* `load_struct`'s own `{unknown_key_for_struct, ...}` check (`elixir_map.erl:204-206`) runs, so the user sees `key :zzz not found` instead of `unknown key :zzz for struct A`. Both are clean compile errors; the security posture (bad key ⇒ compilation fails) is identical. The line-205/131 check stays a live fallback for lenient *custom* `__struct__/1` implementations that silently drop unknown keys.
- **Non-quotable struct defaults** (e.g. a `fun` default) make `elixir_quote:escape/3` (`elixir_map.erl:38`) raise a clean `ArgumentError` outside `load_struct`'s try/catch. This is the struct author's own compile-time code and carries no uplift; a minor robustness/diagnostic nit only.
- **`maybe_load_struct_info/4` is exported** but has no in-tree external callers; treated as internal-only. A future external caller passing a non-`Macro.Env` `E` would `badkey`-crash on the `?key(E, …)` accesses — a contract violation, out of scope.

### Ruled out

**Code execution — `Name:'__struct__'(Assocs)` / `ExternalFun(Assocs)` (`load_struct`, `elixir_map.erl:185/197/199`).**
Reason: `Name` is a compile-time atom validated by `validate_struct/2`; invoking the struct's `__struct__/1` is the by-design mechanism of struct expansion, reachable only via `elixir_expand`, so the caller already has compile-time code execution — no uplift; the return value is fully validated and exceptions are caught and re-raised with an augmented frame.
Axes examined: return non-map, wrong `__struct__` atom, non-atom `__struct__`, struct-fn `throw`, `undef`, unknown-key, non-quotable (`fun`) default | Not examined: none.
Basis: verified (read `elixir_map.erl:175-224`; ran all six hostile `__struct__/1` cases — each gave a clean `CompileError`/typed exception, no internal crash).

**Code execution — `Name:'__info__'(struct)` (`maybe_load_struct_info`, `elixir_map.erl:155`).**
Reason: same as above — `Name` is a validated atom, reached only via expansion, guarded by `try … catch error:undef`; used to fetch struct field metadata at module level.
Axes examined: undefined module (`undef`), self-reference (`module =:= Name`), not-loaded/not-compiling | Not examined: none.
Basis: verified (read `elixir_map.erl:142-165`; ran module-level `@x %A{a:1, zzz:2}` and undefined-struct cases → clean errors).

**Code loading — `code:ensure_loaded(Name)` (`is_compiling_struct`, `elixir_map.erl:235`); `erlang:module_loaded(Name)` (`elixir_map.erl:154`).**
Reason: `Name` is already an atom (no `*_to_atom` in this file, so no atom-table exhaustion), and loading a source-named module at compile time is standard compiler behaviour with no uplift beyond the code execution the source author already has.
Axes examined: atom-creation/exhaustion (none — `Name` pre-exists), nonexistent module, `on_load` side effects | Not examined: none.
Basis: verified (read `elixir_map.erl:234-235,154`; grep confirmed no `list_to_atom`/`binary_to_atom` in file).

**Shared/global state reads — `ets:lookup_element(Set,{elixir,struct},2)` (`elixir_map.erl:170`); `erlang:get(elixir_compiler_info)` (`elixir_map.erl:243`).**
Reason: read-only; the ETS match key `{elixir, struct}` is a compile-time constant (no match-spec injection) and the lookup is wrapped in `try … catch _:_ -> false`; the process-dict read is a boolean presence check.
Axes examined: match-spec injection (constant key), missing table/badarg (caught), unexpected value | Not examined: none.
Basis: verified (read `elixir_map.erl:167-173,242-244`).

**Blocking wait — `wait_for_struct` → `'Elixir.Kernel.ErrorHandler':ensure_compiled/4` (`elixir_map.erl:244`).**
**Promoted to a low-confidence finding** (see Security section). Not left as a bare `asserted` ruled-out: the cycle/deadlock guarantee it relied on lives in `Elixir.Kernel.ErrorHandler`, which is neither a callgraph neighbour nor present in this checkout (re-confirmed this pass: `grep -rl ensure_compiled`, `grep -rln Kernel.ErrorHandler`, `find -iname '*error_handler*'` all empty), so it could not be verified or executed.

**Deserialization/quoting — `elixir_quote:escape(WithoutKeys, escape, false)` (`elixir_map.erl:38`).**
Reason: escapes the struct's own default values (data) into AST with `unquote=false`; no execution, and non-quotable defaults raise a clean `ArgumentError`. Values originate from the struct author's own compile-time code.
Axes examined: `fun`/PID default (clean `ArgumentError`), ordinary literals, large/deep default term (source-bounded) | Not examined: none.
Basis: verified (read `elixir_map.erl:33-39`; ran the `fun` default case → clean `ArgumentError: cannot escape #Function…`).

**Resource — recursion in `validate_match_key/3`, `is_literal/1`, `validate_kv/4` (`elixir_map.erl:73-121`).**
Reason: recursion depth equals source AST nesting depth, which the parser must first build (deeper recursion there), and Erlang's growable per-process stack absorbs it; work is linear in source size. No super-linear blow-up.
Axes examined: 50 000- and 500 000-deep nested match key, nested tuples/lists, improper KV lists | Not examined: none.
Basis: verified (read `elixir_map.erl:73-121`; ran depth 50 000 and 500 000 nested match keys → both compiled without stack failure).

**Resource — `assert_struct_info_if_not_function` O(N·M) key scan (`lists:any` over `Info` per assoc, `elixir_map.erl:131`).**
Reason: quadratic only in `N` (keys in the literal) × `M` (fields the struct declares); both require proportional source, and any unknown key at module level (`function := nil`) raises on the first miss via `function_error → file_error`. Cost is dominated by the linear work of compiling the large struct definition it presupposes; the source author has compile-time code execution regardless — no DoS uplift.
Axes examined: many valid keys × many fields, first-key-unknown early raise, large pre-existing library struct with small literal | Not examined: none.
Basis: verified (read `elixir_map.erl:128-138`; traced early-raise via `file_error` `no_return` semantics; N,M provably source-proportional).

**Validation predicates — `validate_struct/2`, `is_literal/1`, `validate_not_repeated/4`, `validate_kv/4` (`elixir_map.erl:92-126`).**
Reason: these gate diagnostics (repeated key, KV-pair shape, struct-name form), not a security property; wrong answers yield at most a missed warning or a redundant compile error, never acceptance of unsafe code — the bad key/name still reaches expansion and fails there. `validate_kv`'s `not_kv_pair` uses `file_error`, which raises immediately.
Axes examined: non-KV element, repeated literal vs non-literal key, `^var`/variable/atom struct names, non-2-tuple assoc, match vs non-match context | Not examined: none.
Basis: verified (read `elixir_map.erl:92-126`; ran `%{1,2,3}`, repeated-key, `%foo{}`, `%Foo{[a:1]}`, match-key variants → all clean errors/warnings; `lists:nth(Index,Original)` stays in-bounds since `Index ≤ position ≤ length`).

**Template/format — `format_error/1` (`elixir_map.erl:252-305`).**
Reason: every format string is a constant literal; user-influenced values are passed as `~ts` arguments through `'Elixir.Macro':to_string/1` / `elixir_aliases:inspect/1` (no `~`-directive injection), and the output is a compile diagnostic shown to the developer compiling the source (no cross-user boundary).
Axes examined: format-directive injection (constants only), non-string arg to `~ts` (Macro/inspect return charlists), oversized AST (source-bounded) | Not examined: none.
Basis: verified (read `elixir_map.erl:252-305`).

**Controlled re-raise — `erlang:raise(Kind, Reason, Info)` (`elixir_map.erl:223`).**
Reason: faithfully re-throws the struct author's own compile-time exception with an added "expanding struct" frame; no attacker-controlled control-flow beyond the exception the author's own `__struct__/1` already raised.
Axes examined: `throw`/`error`/`exit` from `__struct__`, arbitrary `Reason` term | Not examined: none.
Basis: verified (read `elixir_map.erl:216-224`; ran `throw(:boom)` → propagated as a clean throw).

**Trace hook — `elixir_env:trace({struct_expansion, …}, E)` (`elixir_map.erl:231`).**
Reason: emits a struct-expansion event; the callbacks it invokes are drawn **only** from the env's `tracers` field, which is operator-configured compiler config (trusted per the folder threat model), not from any attacker-supplied value in this file. The event tuple is a constant shape with a validated atom `Name` and the `Keys` list.
Axes examined: event shape (constant tuple), callback source (env `tracers` only) | Not examined: none.
Basis: verified (re-read neighbour `elixir_env.erl:33-34`: `trace(Event, #{tracers := Tracers} = E) -> [ok = Tracer:trace(Event, E) || Tracer <- Tracers]` — it iterates strictly over `E`'s `tracers` list and calls nothing else; `elixir_env:new/0` seeds `tracers => []` at `elixir_env.erl:30`, populated only from compiler config. No attacker-controlled value in `elixir_map` selects a callback). Tracer-callback trust is the threat model's operator-config boundary (out of scope), not an untried input axis.

**`delete_struct_key`/`delete_struct_key_assoc` → `lists:keytake('__struct__',1,Assocs)` (`elixir_map.erl:65`).**
Reason: runs on raw (pre-validation) map args, but `lists:keytake/3` tolerates non-tuple list elements (returns `false`) on this OTP, so non-KV struct bodies do not crash here; they are cleanly rejected downstream by `validate_kv`'s `not_kv_pair`.
Axes examined: non-tuple elements, `__struct__` present/absent, `|`-update RHS, empty list | Not examined: none.
Basis: verified (read `elixir_map.erl:59-71`; ran `lists:keytake(a,1,[1,2,3])` → `false`, and `%Foo{1,2,3}` → clean `expected key-value pairs` error).

### Did I verify this?

- **load_struct dispatch / `__info__` / escape / re-raise / delete_struct_key / validations / recursion / keytake** — all `verified`: read the code paths and *ran* the corresponding hostile inputs (non-map/wrong-name/non-atom/throwing/fun-default/unknown-key `__struct__`; `%{1,2,3}`; repeated keys; 500 000-deep nesting; `lists:keytake` on non-tuples). `Not examined: none` for each.
- **`code:ensure_loaded`/`module_loaded`/ETS/proc-dict reads** — `verified`: read the calls and confirmed by grep that the file creates no atoms and uses a constant ETS key inside a `try`. `Not examined: none`.
- **`assert_struct_info_if_not_function` O(N·M)** — `verified`: read `elixir_map.erl:128-138`, traced the early-raise via `file_error` `no_return`, both N and M provably source-proportional; no untried axis.
- **`format_error`** — `verified`: constant format strings, arguments are the only variable, output is a same-user compile diagnostic.
- **`elixir_env:trace/2`** — `verified`: re-read the neighbour `elixir_env.erl:33-34` and `:30` this pass. The cross-file question (where do trace callbacks come from?) is structural, not a differential guard, and the read resolves it definitively: `trace/2` iterates only `E`'s `tracers` list, seeded empty and filled from compiler config. The residual "is a configured tracer trusted?" is the threat model's operator-config boundary, out of scope — not an untried input axis.
- **`wait_for_struct`/`ensure_compiled`** — **could not verify** (unchanged from the prior upgrade). Re-confirmed this pass that `Elixir.Kernel.ErrorHandler` is not a callgraph neighbour and returns zero files across the checkout (three separate searches), so its cycle-breaking guarantee can be neither read nor executed here. Recorded as a **low-confidence DoS finding**, not a silent assertion. A cross-file pass with access to the error-handler source should confirm `ensure_compiled/4` returns a terminal status on a compilation cycle, which would downgrade it back to ruled-out.

### Changes from the prior pass

- **No status changes.** The two upgrades made in the previous hedge-resolution pass are preserved and re-confirmed:
  - `wait_for_struct` blocking wait remains a **low-confidence finding** — re-verified this pass that the dependency module is absent from the checkout and outside the neighbour set, so the assumption cannot be cleared. Stays in the safe (unverified → finding) direction.
  - `elixir_env:trace/2` remains **verified** — the neighbour was re-read this pass (`elixir_env.erl:30,33-34`) and the routing claim holds.
- No ruled-out entry moved from `verified` toward `asserted`, and none moved from a finding back to a dismissal. Every axis marked `Not examined: none` was executed in the prior pass; the file is byte-identical (sha256 unchanged), so those executed results still hold.

### Dropped in re-review

None. Every ruled-out entry, caveat, and finding from the prior report is preserved. No dismissal was strengthened toward `verified` without executed evidence, and the sole unverifiable cross-file dependency stays a low-confidence finding.
