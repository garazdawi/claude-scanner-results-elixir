# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_compiler.erl
<!-- sha256: e09147ddb640a561840d130f6d6ead2c14212cd4afefe2a48f6817eaecbef398 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_compiler.erl
<!-- sha256: e09147ddb640a561840d130f6d6ead2c14212cd4afefe2a48f6817eaecbef398 -->

_Re-reviewed on 2026-08-28_

# Security review — `elixir_compiler.erl` (re-review, hedge pass)

## Summary
This file is the compiler front-end that orchestrates turning Elixir quoted forms into loaded BEAM modules (`string`/`quoted`/`file`/`compile`/`interpret`). Every dangerous sink it contains is either by-design code execution on caller-trusted forms or operates on compiler-generated (internal) data. This pass re-examined the two dismissals the first-pass reviewer flagged for hedge phrasing ("out of scope for this file") — `string/3`'s tokenizer routing and `optimize_defmodule`'s config reads. I re-read every neighbour I cite (`elixir.erl:82-105`, `456-526`, `346-372`; `elixir_utils.erl:136-155`) and confirmed each citation against the live source. Both flagged entries rest on a residual axis that reading a neighbour cannot clear — a **cross-file behavioural dependency** (`string/3`) and an **out-of-scope operator-config axis** (`optimize_defmodule`) — and neither residual is a defect in *this* file, so both remain justified `asserted` dismissals rather than becoming findings. **No security defect found.**

## Security

No findings.

Every code-execution path in this file (`interpret/3` → `elixir:erl_eval`; `compile/4`/`spawned_compile/3` → `elixir_erl_compiler:noenv_forms` + `code:load_binary` + `Module:Fun(Args)`) is the documented purpose of a compiler: compiling/evaluating Elixir *is* running it, and the folder threat model explicitly places "treating an untrusted quoted AST as safe to eval" on the caller, not on this module. The only crash-prone parsers (`is_purgeable/1`, the `{ok, Bin}` match in `file/2`) operate on internal/compiler-generated data or a developer-controlled path, not on attacker-influenced bytes. There is no `binary_to_term`, no `os:cmd`/`open_port`, no crypto, no network, and no path built from untrusted components.

### Ruled out

**`interpret/3` → `elixir:erl_eval(ErlExprs, Binding, E)` (line 59) — code execution.**
Reason: evaluates translated quoted forms; per the folder contract the caller is responsible for only evaluating trusted forms — this is `Code.eval_quoted`-class behaviour, i.e. running code is the documented purpose, not a bug. Confirmed against the callee: `elixir:erl_eval/3` (`elixir.erl:346-372`, re-read this pass) is a thin wrapper over `erl_eval:expr/4` with save/restore of the `?elixir_eval_env` process-dictionary key — unambiguously an evaluator, no sandbox contract.
Axes examined: forms origin (caller quoted AST), exception propagation (re-raised with augmented stacktrace, lines 61-62), callee semantics (`erl_eval:expr`, read `elixir.erl:346-372`) | Not examined: none.
Basis: verified. Confidence: high.

**`compile/4` + `spawned_compile/3` + `dispatch/4` → `elixir_erl_compiler:spawn`, `code:load_binary`, `Module:Fun(Args)` (lines 73, 87, 102) — code execution / code loading.**
Reason: compiling Elixir source loads and runs its module bodies at compile time — the documented contract of `Code.compile_string`/`compile_quoted`; feeding untrusted source here is arbitrary code execution by design, not a defect in the machinery.
Axes examined: forms origin (expander output), dispatch target (retrieved compiler-pool module, fixed wrapper), args (`list_to_tuple(ArgsList)`, internal) | Not examined: none.
Basis: verified. Confidence: high.

**`string/3` → `elixir:'string_to_quoted!'(Contents, 1, 1, File, elixir_config:get(parser_options))` (line 11) — untrusted source parsing.**
Reason: `Contents` is attacker-controllable, but this file performs **no byte-level processing** — it forwards the string verbatim to `elixir:'string_to_quoted!'/5`. I re-read the full routing chain this pass to reconfirm no bypass: `'string_to_quoted!'/5` (`elixir.erl:518-526`) → `string_to_quoted/5` (`elixir.erl:500-516`) → `string_to_tokens/5` → `elixir_tokenizer:tokenize/4` (`elixir.erl:456-462`, line 457 is the unconditional tokenize call), then `tokens_to_quoted/3` → `elixir_parser:parse` (`elixir.erl:464-479`). The string is unconditionally routed through the tokenizer; there is no raw/`Interpol=false`/"skip tokenizer" path reachable from this file. The Unicode-rejection *behaviour* (bidi/NFC/mixed-script/escape/grapheme handling, checklist claims 1-12) is owned by `elixir_tokenizer.erl`/`elixir_interpolation.erl`.
Axes examined: sanitiser routing — this file forwards to the tokenizer path with no bypass (re-verified by reading `elixir.erl:456-526`, line 457) | Not examined: the Unicode-rejection axes themselves (does the tokenizer actually reject bidi / non-NFC / mixed-script / bad-escape input). This is a downstream behavioural axis owned by another file; per the re-review rules, reading the routing does not clear it, and executing an input through `elixir_tokenizer:tokenize/4` would test the *tokenizer's* behaviour, not this file's forwarding — so it cannot be cleared from within this file's review. It stays on `Not examined`.
Basis: asserted. Confidence: high (in the dismissal). Depends on: `elixir_tokenizer.erl` / `elixir_interpolation.erl` — `string_to_quoted!` (via `elixir_tokenizer:tokenize`, `elixir.erl:457`) must reject malformed UTF-8, bidi controls, non-NFC/mixed-script identifiers and bad escapes before forms reach this module.
Not upgraded to a finding: (a) the routing to the tokenizer is confirmed with no bypass in this file; (b) the threat model assigns byte-sanitisation to the tokenizer as a cross-file invariant; (c) the residual is a cross-file dependency on another module's behaviour, not a defect in `elixir_compiler.erl`. The first-pass hedge "is a property of that module's review" is retained as an explicit `Depends on:` cross-file dependency, which is the correct classification, not a hedge about this file.

**`file/2` → `{ok, Bin} = file:read_file(File)` then `elixir_utils:characters_to_list(Bin)` (lines 33-34) — file read + UTF-8 decode.**
Reason: `File` is a developer/build-supplied path (`Code.compile_file`-class), not attacker-chosen; the `{ok,Bin}` hard match only turns a read error into a badmatch crash for a path the caller already controls. Invalid/incomplete UTF-8 in the content is surfaced as a proper `UnicodeConversionError` (re-read `elixir_utils.erl:138-155`: both `{error,...}` and `{incomplete,...}` route to `conversion_error/3` which `error/1`s), not silent garbage — satisfying checklist claim UTF8_DECODE_ERRORS_SURFACED for this call site.
Axes examined: content encoding (invalid + incomplete UTF-8, both raise — read `elixir_utils.erl:136-155`), read-failure (badmatch on caller-controlled path) | Not examined: path axes (traversal/symlink) — the path is not an untrusted component here.
Basis: verified. Confidence: high.

**`quoted/3` → process-dictionary `elixir_module_binaries` save/restore (lines 15, 18, 27, 29) — shared mutable state.**
Reason: state is process-local, saved to `Previous` and restored in `after`; the sole writer guards on `is_list(Current)` so a restored `undefined` value cannot crash or leak across nested evals.
Axes examined: nesting/reentrancy (correct save→[]→restore), non-list restored value (guarded no-op), callback throwing (restored via `after`) | Not examined: none.
Basis: verified (read `elixir_module.erl:609-615` in first pass; save/restore structure re-confirmed here at lines 15-30). Confidence: high.

**`is_purgeable/1` + `do_is_purgeable/1` → BEAM chunk parsing with `<<_:(4*trunc((Size+3)/4))/binary, Rest/binary>> = Beam` (lines 90-98) — deserialisation/parsing badmatch.**
Reason: the `Binary` parsed here is the output of `compile:noenv_forms` (via `elixir_erl_compiler:noenv_forms`, line 86), i.e. always a well-formed FOR1/BEAM produced by the Erlang compiler for a fixed wrapper module; it never sees attacker-influenced bytes, and even a wrong result only affects compiler-pool reuse inside the already-running compile process.
Axes examined: real compiler BEAM (returns `true`), truncated chunk (badmatch), non-FOR1 garbage (function_clause), origin (compiler-generated) | Not examined: none.
Basis: verified (ran the parser against real/truncated/garbage input in first pass — see Validation). Confidence: high.

**`code_mod/6` → `elixir_utils:characters_to_list(Relative)` and abstract-form construction (lines 110-124) — template/AST building.**
Reason: builds Erlang abstract-format `file`/`module`/`export`/`function` nodes; `Relative` (source path) is placed as a literal `file` attribute string (line 115) and as an `elixir_to_erl` literal (line 123), not spliced into a format string or evaluated, so there is no injection target.
Axes examined: path as literal attribute (no format interpretation), no code-string interpolation | Not examined: none.
Basis: verified (read lines 110-124). Confidence: high.

**`bootstrap/0` / `bootstrap_file/2` / `binary_to_path/2` → `file:read_file`, `file:write_file(Path, Binary)`, `filename:join`, `atom_to_list(ModuleName)` (lines 167-193, 246-251) — file writes / atom-to-path.**
Reason: reachable only via `elixir_compiler:bootstrap/0`, the build-time bootstrapper for Elixir's own stdlib; paths (`lib/elixir/lib`, `lib/elixir/ebin`) are constants and module names come from compiling Elixir's own trusted sources — no runtime/attacker input path.
Axes examined: caller (build-only entry), path components (compile-time constants + own module names) | Not examined: none.
Basis: verified (read lines 167-251). Confidence: high.

**`optimize_defmodule/2` config reads + `retrieve_compiler_module`/`return_compiler_module` (lines 39-40, 82, 127, 130) — shared config / gen_server pool.**
Reason: `elixir_config:get(parser_options)` (line 11), `get(tracers)` (line 19) and `is_bootstrap()` (line 40) are operator-controlled compiler config, explicitly out of scope in the threat model. I re-read the config-seeding site in the callgraph neighbour this pass and reconfirmed the provenance: these keys are seeded as static application defaults at OTP `start/2` — `{parser_options, [{columns, true}]}`, `{tracers, []}` (`elixir.erl:98,100`) and `bootstrap => false` via `elixir_config:static/1` (`elixir.erl:104`), plus the bootstrap-time overrides in `elixir_compiler:bootstrap/0` (lines 169-176). None of these values is derived from parsed source. The compiler-module pool calls (`retrieve_compiler_module`/`return_compiler_module`) are internal `elixir_code_server` requests whose payload is a module atom + boolean.
Axes examined: config origin — confirmed operator/application-seeded static defaults, not source-derived (re-read `elixir.erl:82-105` and `elixir_compiler.erl:169-176`); pool message payload (module atom + boolean) | Not examined: whether an operator can inject a malicious tracer/parser_option. This axis is out of scope by the threat model (an operator controlling compiler config already has code execution) and is not an executable differential I can clear from this file, so it remains on `Not examined`.
Basis: asserted. Confidence: high (in the dismissal). Depends on: `elixir_config.erl` / `elixir_code_server.erl` — the config table is written by `elixir_config:static/1`/`new/1` at application start (seeding read in `elixir.erl:82-105`), and the compiler-module pool is populated by `elixir_code_server`, not from parsed source.
Not upgraded to a finding: the only residual axis is operator-controlled config, which the threat model places out of scope. The first-pass phrase "out of scope by threat model" is not a hedge about *this* file's behaviour — it is the correct threat-model scoping, backed by the confirmed static-default provenance above.

**`no_tail_optimize/2` / `expand_defmodule/2` → hygiene variable `{result, Meta, ?MODULE}` and alias/module expansion (lines 141-163) — macro hygiene / dispatch.**
Reason: the injected `result` variable carries context `elixir_compiler` (`?MODULE`, lines 160/162), disjoint from user-code contexts, so it cannot capture or be captured; alias expansion (`elixir_aliases:expand_or_concat`, line 146) and `elixir_module:compile` (line 156) are the ordinary, by-design compile path.
Axes examined: variable context collision (distinct module context), block substitution (user block nested, not evaluated here) | Not examined: none.
Basis: verified (read lines 139-163). Confidence: high.

### Did I verify this?

- `interpret/3`, `compile/4`/`spawned_compile/3`/`dispatch/4`, `quoted/3`, `file/2` decode, `is_purgeable/1`, `code_mod/6`, bootstrap paths, `no_tail_optimize`/`expand_defmodule` — all `verified`; `Not examined` is `none`. Cited neighbour lines re-read and confirmed live this pass: `elixir.erl:346-372` (erl_eval wrapper), `elixir_utils.erl:136-155` (decode-error surfacing). `is_purgeable` exercised directly in the first pass (Validation below).
- `string/3` — kept `asserted`. The **routing** axis is verified (`elixir.erl:456-526`, line 457: the string is forwarded unconditionally through `elixir_tokenizer:tokenize` with no bypass in this file). The **Unicode-rejection behavioural** axes stay on `Not examined`: they are owned by `elixir_tokenizer.erl`/`elixir_interpolation.erl`, and the re-review rules are explicit that reading a neighbour never clears an axis and that only trying the axis on *this file's* implementation would — but this file has no such implementation to try (it only forwards). So the entry stays in the safe `asserted` direction, carried as an explicit cross-file `Depends on:`, not a finding. This is a downgrade-safe classification of the flagged hedge, not an upgrade.
- `optimize_defmodule` config / compiler-pool reads — kept `asserted`. The config **provenance** axis is verified as static application/operator defaults (`elixir.erl:82-105`), not source-derived. The sole residual axis — a malicious operator injecting config — is out of scope by the threat model and is not an executable differential, so it stays `Not examined` and the entry stays `asserted`.

**Validation (is_purgeable robustness / origin):**
`ran erlc + erl` (first pass) on an extracted copy of `is_purgeable/do_is_purgeable`:
```
real BEAM is_purgeable = true (size 456)
truncated BEAM is_purgeable = {error,{badmatch,<<...>>}}
non-FOR1 garbage is_purgeable = {error,function_clause}
```
Crashes occur only for malformed BEAM; the real input is always `compile:noenv_forms` output (via `elixir_erl_compiler:noenv_forms`, line 86), so the crash branches are unreachable with real data.

## Risks / caveats
- The `.ex` library callers (`Code.compile_string/2`, `Code.compile_file/1`, `Kernel.ParallelCompiler`) that reach `string/3`/`quoted/3`/`file/2` are **not present in this `src/` checkout**, so I could not read those exact call sites; the reachability of these entry points from documented public APIs is inferred from their names/roles and the folder threat model rather than read.
- The single retained cross-file dependence is on the tokenizer/interpolation modules sanitising raw source before forms reach this module (`string/3`'s `Depends on:`). This review confirmed only that `elixir_compiler` routes source to `elixir_tokenizer:tokenize` with no bypass in this file; it did **not** execute the tokenizer's bidi/NFC/mixed-script/escape rejection, because that behaviour belongs to the tokenizer's own review. If that gate were ever bypassed inside the tokenizer (e.g. a future sigil/`Interpol=false` path), untrusted forms could reach the by-design eval/compile sinks here — but that would be a defect in the tokenizer, not in `elixir_compiler.erl`.
- Nothing in this file defends against a malicious *operator* who controls `tracers`/`parser_options`/bootstrap config; that is explicitly out of scope (such an operator already has code execution). The config values are confirmed operator/application-seeded static defaults (`elixir.erl:82-105`), not source-derived.

## Dropped in re-review
None. Every dismissal from the prior report is preserved. The two flagged `asserted` entries (`string/3`, `optimize_defmodule` config) remain `asserted`. Per the re-review rules, moving them to `verified` would require trying the untried axis on this file's own implementation — which does not exist for either (both residuals live in other files / out of scope), and reading the neighbours (which I did, and re-confirmed live) cannot clear an axis. Upgrading either to a finding would require executed evidence of a defect in *this* file, of which there is none. Keeping them `asserted` — with the flagged hedge phrasing reclassified as an explicit cross-file `Depends on:` (`string/3`) and an explicit out-of-scope threat-model scoping (`optimize_defmodule`) — is the safe direction the rules prescribe.
