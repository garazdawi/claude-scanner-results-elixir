# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_compiler.erl
<!-- sha256: 48aafb3f9708f21c0e73d381617898b717d094dd578aa1596def87ef1c358b9d -->

_Analysed on 2026-08-27T16:47:27Z_

## Summary

`elixir_erl_compiler.erl` is trusted compiler-driver glue that runs the Erlang back-end (`v3_core`/`compile`) over already-expanded, compiler-generated forms and marshals diagnostics between a spawned worker and its parent; every sink draws on compilation-internal data (forms, dev-controlled file paths, operator options), so I found **no security findings**.

## Security

No findings.

Every security-relevant sink in this file is fed by values that originate inside the trusted compilation pipeline. The realistic attacker in the folder threat model is "a supplier of untrusted Elixir source." That source only reaches this module *after* `elixir_expand:expand` (arbitrary macro code has already executed) and *before* the compiled module runs — so no sink here confers a capability the attacker doesn't already have from the fact that their code is being compiled/run. None of the folder's Unicode obligations (bidi, newlines, NFC, mixed-script, escapes, grapheme) apply to this file; the only Unicode-adjacent call (`characters_to_list(File)`) properly surfaces conversion errors (verified below). There is no `binary_to_term`, no atom creation, and no OS/file/network/crypto sink.

## Risks / caveats

- The audit's dismissals depend on two out-of-file contracts, both consistent with the callgraph: (1) the `Module` atom in `{Line, Module, Desc}` diagnostic tuples is a compiler-pass name emitted by the Erlang compiler (`v3_core`/`compile`/`erl_lint`/`sys_core_fold`/`beam_validator`), never derived from source content, so the `Module:format_error(Desc)` dynamic dispatch in `custom_format/2` is not attacker-influenced; (2) `Opts` carrying `{parse_transform, _}` come from `@compile`/`env_compiler_options()`, which is developer/operator-controlled and equivalent in power to the macro execution that has already happened. If either contract were violated by an in-folder caller, sinks #7 and #12 below would need re-rating — but the callers (`elixir_erl.erl:525`, `elixir_compiler.erl:73,86`; `elixir_module.erl:169`) pass compiler-generated forms and env-derived options only.
- `spawn/1`'s `receive` blocks forever on a child DOWN reason that is neither `{ok,_,_}` nor a 4-tuple (e.g. `killed`). This is reachable only by an external `exit(ChildPid, kill)`; the child Pid is never exposed and a co-tenant process able to signal it is out of scope per the folder threat model. Robustness nit, not a finding.
- File paths passed as `File` are developer-controlled source paths; a malicious *operator* controlling `ERL_COMPILER_OPTIONS`/`@compile`/tracers already has code execution and is explicitly out of scope.

### Ruled out

- **`spawn_monitor(Fun)` (spawn/1, line 20)** — Code execution.
Reason: `Fun` is a compiler-internal closure supplied by `elixir_compiler:compile/4` (`fun()->spawned_compile(...)`) and `elixir_module.erl:169`; never attacker-supplied.
Axes examined: caller identity, closure origin | Not examined: none.
Basis: verified — read all three in-folder call sites.

- **`erlang:raise(Kind, Reason, Stack)` (spawn/1, line 39)** — Code execution / error propagation.
Reason: re-raises the child's own caught exception (Kind/Reason/Stack produced by the worker's `try/catch`); standard cross-process error propagation, no external term injection.
Axes examined: reason provenance, tuple shape | Not examined: none.
Basis: verified — read lines 25–40; the only `exit` sites are the 3-tuple and 4-tuple forms.

- **`receive` DOWN-reason match (spawn/1, lines 33–40)** — Concurrency / blocks-forever.
Reason: a non-matching reason (`killed`) requires an external kill of the un-exposed child Pid (co-tenant, out of scope); worker code only ever exits `{ok,_,_}`/`{Kind,_,_,_}`, both of which match.
Axes examined: normal exit, caught-failure exit, external kill | Not examined: none.
Basis: verified — `ran /tmp/t.erl`: `ok-exit -> match_ok_clause`, `fail-exit -> match_err_clause`, `kill-exit reason=killed -> no_match_would_block`.

- **Process-dictionary writes `put(elixir_compiler_info | elixir_code_diagnostics, …)` (spawn/1 lines 22–23; copy_diagnostics line 47)** — Shared mutable state.
Reason: writes are process-local (worker's own dict, then parent's own dict); no cross-tenant or global state, values are compiler info / diagnostic lists.
Axes examined: scope (process-local vs global), value provenance | Not examined: none.
Basis: verified — read lines 9–48.

- **`Head ++ Tail` diagnostics concat (copy_diagnostics, line 47)** — Resource consumption.
Reason: cost is O(number of diagnostics), bounded by warnings/errors actually produced by compilation; no attacker amplification beyond inherent compile cost.
Axes examined: list length source, nesting/recursion depth | Not examined: none.
Basis: verified — read lines 42–48; `Head` is the worker's accumulated diagnostics list.

- **`persistent_term:put(?MODULE, Options)` (env_compiler_options/0, line 54)** — Shared mutable state (VM-global).
Reason: fixed key, written at most once (memoised behind the `undefined` check); value is `compile:env_compiler_options()` derived from `ERL_COMPILER_OPTIONS`, an operator-controlled env var (out of scope). No repeated-write global-GC storm.
Axes examined: key cardinality, write frequency, value origin | Not examined: none.
Basis: verified — read lines 50–59.

- **Parse-transform execution `compile:noenv_forms(Forms, [… parse_transform …])` (erl_to_core/2, line 72)** — Code execution.
Reason: the `parse_transform` path fires only when `Opts` contains `{parse_transform, _}`, which comes from `@compile`/compiler options (developer/operator-controlled) and is equivalent to the compile-time macro execution that already ran; supplies no new capability.
Axes examined: Opts provenance, trigger predicate (`requires_abstract_format`) | Not examined: whether any in-folder caller forwards a source-derived `parse_transform` value.
Basis: asserted.
Depends on: `elixir_erl.erl` / `elixir_compiler.erl` — `Opts` reaching `erl_to_core/noenv_forms` carry only `@compile`/`env_compiler_options()` entries, not source-string-derived module atoms.

- **`{ok, DbgForms} = sys_coverage:beam_debug_info(Forms)` badmatch (erl_to_core/2, line 68)** — Dead/robustness.
Reason: only reached when `beam_debug_info` is in `Opts` (a debug build option); a non-`{ok,_}` return crashes the compile with a badmatch — a robustness failure on a dev/operator-selected path, no attacker uplift.
Axes examined: option gating, return-shape | Not examined: none.
Basis: verified — read lines 63–70 and `requires_abstract_format/2` (lines 78–85).

- **`elixir_utils:characters_to_list(File)` (noenv_forms/3, line 88) and `characters_to_binary(File)` (format_errors/warnings, lines 135,144)** — UTF-8 decode / template input.
Reason: `File` is the developer-controlled source path; the helper raises `conversion_error` on invalid/incomplete UTF-8 rather than emitting replacement garbage — obligation UTF8_DECODE_ERRORS_SURFACED satisfied.
Axes examined: invalid byte, incomplete tail, valid input | Not examined: none.
Basis: verified — `ran unicode:characters_to_list`: `invalid: {error,[],…}`, `incomplete: {incomplete,"ab",…}`, `valid: "ok"`; read `elixir_utils.erl:136–158`.

- **`compile:noenv_forms(CoreForms, CompileOpts)` (noenv_forms/3, line 96)** — Code execution / compilation.
Reason: compiles the module's own Core Erlang forms; this *is* the compilation the caller asked for, over compiler-generated forms. No sandbox contract to violate.
Axes examined: forms provenance, options provenance | Not examined: none.
Basis: verified — read lines 87–118 and caller `elixir_compiler.erl:78–88`.

- **`io_lib:format(Prefix ++ "…", Args)` (incompatible_options/3, line 121)** — Template / format-string injection.
Reason: `Prefix` is one of two constant string literals (`"could not compile module ~ts"` / `"could not compile module"`) passed by in-file callers (lines 102,105,112); no attacker-controlled format directives.
Axes examined: format-string source, arg count/type | Not examined: none.
Basis: verified — read lines 101–128.

- **`custom_format(Module, Desc) -> Module:format_error(Desc)` dynamic dispatch (custom_format/2, line 206)** — Code execution (apply of external module atom).
Reason: `Module` is the compiler-pass name inside the Erlang compiler's `{Line, Module, Desc}` diagnostic tuple (e.g. `erl_lint`, `sys_core_fold`, `v3_core`, `beam_validator`), emitted by `compile`/`v3_core`, not derived from source text; worst case on an unexpected module is an `undef` crash, not attacker-directed dispatch.
Axes examined: Module atom origin, missing-`format_error` failure mode | Not examined: whether some compiler pass can be induced to emit a source-controlled Module atom.
Basis: asserted.
Depends on: OTP `compile`/`v3_core`/`beam_validator` — the `Module` field of returned error/warning tuples is the diagnosing pass's own module name, independent of the compiled source content.

### Did I verify this?

- `spawn_monitor`, `erlang:raise`, DOWN-reason match, process-dict writes, diagnostics concat, `persistent_term:put`, `sys_coverage` badmatch, `characters_to_list/binary`, `compile:noenv_forms`, `io_lib:format` prefix — all **verified**: I read the exact lines, traced the boundary to in-folder callers, and where a behavioural claim was involved I ran it (`/tmp/t.erl` for the DOWN-reason matching; `unicode:characters_to_list` for conversion-error surfacing). Their `Not examined` lists read `none`; no `Depends on` remains.
- **Parse-transform execution** — kept **asserted**. Reading this file confirms the trigger predicate, but confirming that *no* in-folder caller ever forwards a source-string-derived `parse_transform` atom requires the caller-side contract. `Depends on: elixir_erl.erl / elixir_compiler.erl — Opts carry only @compile/env options`. Even if violated, the resulting capability (compile-time code execution) is one the source-compilation attacker already possesses, so the ceiling is Low regardless.
- **`Module:format_error/1` dynamic dispatch** — kept **asserted**. This file cannot prove the Erlang compiler never emits a source-controlled `Module` atom in a diagnostic tuple; that is a property of `compile`/`v3_core`/`beam_validator`. `Depends on: OTP compiler back-end — diagnostic `Module` field is the pass name, not source-derived`. This is the one dependency a cross-file/OTP-level pass should confirm; I judge it solid because diagnostic tuples are constructed by the passes themselves, but it is not verifiable from within this folder.

