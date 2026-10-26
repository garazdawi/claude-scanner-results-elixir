# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_import.erl
<!-- sha256: fd10160271dafb47711fb0f4c5e03ffe5ed8d4d86c81aa867cd036edee1d6292 -->

_Analysed on 2026-08-27T17:05:34Z_

# Security review — `elixir_import.erl`

## Summary
`elixir_import` is a compile-time import-resolution helper reached only during macro expansion (`elixir_expand`/`elixir_dispatch`/`elixir_module`), and every security-relevant sink in it is either by-design import behaviour on an already-loaded module or an internal-state operation that grants an attacker no capability beyond the arbitrary compile-time code execution that reaching this module already implies — **No findings**.

## Security

No findings.

The whole module operates strictly inside the expansion/compilation phase. I verified that the documented parse-only public API (`Code.string_to_quoted`) does **not** reach import processing (4000-entry `only:` list parsed in 3.34 ms with no import warnings), whereas `Code.compile_string`/`Code.eval_string` do — and those are documented to execute arbitrary code. Consequently any DoS, crash, or state effect inside this file is subsumed by the arbitrary compile-time code the attacker already runs; there is no trust boundary at which a new capability is gained. The module also contains no `binary_to_term`, no `list_to_atom`/`binary_to_atom`, and no file/OS/network/crypto/NIF sinks.

## Risks / caveats
- The dismissals below depend on the invariant that `elixir_import` is only ever entered from the expansion phase (confirmed via the in-folder callgraph: callers are `elixir_expand`, `elixir_dispatch`, `elixir_module`, all expansion-time). If a future caller invoked `import/6` or `format_error/1` from a pure-parse path, the "attacker already runs code" argument would weaken the resource-consumption and diagnostic sinks from *no uplift* to *Low DoS/robustness* — but none exists today.
- `import/6` builds `fun Ref:'__info__'/1` and (fallback) calls `Module:module_info(exports)`. This relies on `elixir_expand.erl:152-155` asserting `is_atom(ERef)` and calling `elixir_aliases:ensure_loaded/3` before dispatch, and on `__info__`/`module_info` being compiler/BEAM-generated metadata accessors rather than arbitrary user code. Both hold in the current tree.

### Ruled out

- **Code execution via `fun Ref:'__info__'/1` / `InfoCallback(functions|macros)` / `Module:module_info(exports)`** (lines 16, 101/106/111/116, 173, 175)
  - Reason: `Ref` is a source-named module that `elixir_expand` has already validated as an atom and `ensure_loaded`; the callbacks invoked are compiler/BEAM-generated metadata accessors, and importing a module is by-design — reachable only in the compile phase where the attacker already has code execution.
  - Axes examined: non-atom `Ref`, unloaded module, module without `__info__` (undef→`module_info` fallback), phase-of-reachability | Not examined: none
  - Basis: verified

- **ETS write `ets:insert(Set, {{import, Tuple}, Receiver})` in `record/4`** (line 151)
  - Reason: `Set` comes from `elixir_module:data_tables(Module)` where `Module` is the *currently-compiling* module (`elixir_dispatch.erl:62` passes `?key(E, module)`), so it writes only to its own compilation state; `error:badarg` is caught. No cross-tenant write and no untrusted term reaches a match spec.
  - Axes examined: table ownership (own module), missing/closed table (badarg caught), `Tuple` shape, cross-module poisoning | Not examined: none
  - Basis: verified

- **ETS read `ets:lookup_element(Set, {import, Pair}, 2)` in `ensure_no_local_conflict/3`** (line 163)
  - Reason: read of the compiling module's own table (caller `elixir_module.erl:187`); `error:badarg` (missing key = the normal "no conflict" case) is caught and used as control flow — no crash, no external influence on the lookup key beyond the module's own definitions.
  - Axes examined: missing key (badarg→false), absent table, key shape, `Elixir.Kernel` short-circuit | Not examined: none
  - Basis: verified

- **O(n²) processing: `ensure_no_duplicates/5`, `intersection/2`, `OldImports -- Except`** (lines 226-235, 275-281, 130-131)
  - Reason: super-linear cost is driven by the length of a source-controlled `only:`/`except:` keyword list, but the path is entered only during compilation/eval, where the attacker's own compile-time code runs arbitrarily; the DoS grants no uplift. Measured cost is also mild (8000 entries ≈ 149 ms).
  - Axes examined: input length scaling (2k/4k/8k), duplicate entries, negative/huge arity values, parse-vs-compile reachability | Not examined: none
  - Basis: verified

- **`format_error/1` diagnostic with `~s` + `'Elixir.Macro':to_string(Value)`** (lines 250-257)
  - Reason: `Macro.to_string/1` returns a UTF-8 **binary**, and `io_lib:format("~s", [Binary])` never raises on a binary (all bytes 0–255); the end-to-end invalid-`:only` path with a Unicode atom produced a clean `CompileError`, not a compiler crash. (A `~s` crash only occurs on a *charlist* with codepoints > 255, which this path never supplies.)
  - Axes examined: binary vs charlist argument, codepoints > 255, Unicode atom value, terminal-escape/mojibake, format-string injection (format is a constant) | Not examined: none
  - Basis: verified

- **`ensure_keyword_list/1` accepts negative/oversized arity (`is_integer(Value)`)** (line 210)
  - Reason: an out-of-range arity simply never matches any real export in `intersection`/`--`, so the entry is dropped; there is no downstream numeric sink. Validation-predicate whose only "wrong answer" is a no-op.
  - Axes examined: negative arity, huge arity, non-integer (rejected), non-atom key (rejected) | Not examined: none
  - Basis: verified

- **`special_form/2` table + `ensure_no_special_form_conflict/5`** (lines 215-224, 298-335)
  - Reason: this is a developer-assistance predicate that discards imports colliding with special forms and warns; it is not a security control, and a wrong answer only affects a compile-time warning/discard, not any trust decision.
  - Axes examined: name/arity coverage of the table, warn-vs-error behaviour, filter fall-through | Not examined: none
  - Basis: verified

- **`'Elixir.Kernel'` conflict-detection short-circuit in `ensure_no_local_conflict/3`** (line 157)
  - Reason: skips the local-vs-import *conflict warning* when the module *being defined* is named `Kernel`; conflict detection is developer assistance, not an authorization boundary, so suppressing it yields no security effect.
  - Axes examined: module-name spoofing (`defmodule Kernel`), which module the guard matches (definee, not import target), effect of skipping (lost warning only) | Not examined: none
  - Basis: verified

### Did I verify this?

- Code-execution sinks — **verified**. Read lines 16/171-176 and callers `elixir_expand.erl:152-155` (atom assertion + `ensure_loaded`); confirmed the callbacks are metadata accessors and the path is compile-phase-only. `Not examined: none`; no `Depends on:` needed.
- ETS write `record/4` — **verified**. Read `elixir_dispatch.erl:62` confirming `Module` is the compiling module; badarg caught (line 154). `Not examined: none`.
- ETS read `ensure_no_local_conflict/3` — **verified**. Read `elixir_module.erl:187` caller and the try/badarg control-flow. `Not examined: none`.
- O(n²) processing — **verified** by running: parse-only 3.34 ms (no import processing) vs compile scaling 2k→8k = 38→149 ms, establishing both the phase boundary and the mild cost. `Not examined: none`.
- `format_error` `~s` — **verified** by running: `io_lib:format("~s", [<<utf8>>])` → `"café_αβ"` (ok), charlist>255 → badarg (not this path), and full `Code.compile_string` invalid-`:only` → clean `CompileError`. `Not examined: none`.
- `ensure_keyword_list` arity range, `special_form` table, `Elixir.Kernel` skip — **verified** by reading the clauses and tracing that each wrong answer is a compile-time no-op/warning with no downstream security sink. `Not examined: none`.

All ruled-out entries are `verified`; no `asserted` dismissals remain, so there are no outstanding `Depends on:` contracts for the cross-file pass.

