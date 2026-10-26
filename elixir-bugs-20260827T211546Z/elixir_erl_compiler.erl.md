# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_compiler.erl
<!-- sha256: 48aafb3f9708f21c0e73d381617898b717d094dd578aa1596def87ef1c358b9d -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_erl_compiler.erl
<!-- sha256: 48aafb3f9708f21c0e73d381617898b717d094dd578aa1596def87ef1c358b9d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (OTP 29 sandbox available; axes executed)_

# Bug-hunting review — `elixir_erl_compiler.erl`

## Spec-totality checklist (exports)

- `spawn/1` — spec `(fun(() -> any())) -> any()`. Body wraps `Fun()` in `try/catch Kind:Reason:Stack`, re-exiting all three exception classes; the only unhandled child-death shape is an untrappable external kill (see finding). Accepts everything the spec declares.
- `noenv_forms/3` — spec `([erl_parse:abstract_form()], binary(), list()) -> {module(), binary()}`. Guard `is_list(Forms), is_list(Opts), is_binary(File)` matches the spec; all non-`{Module,Binary}` outcomes raise rather than returning, so the declared return shape holds.
- `erl_to_core/2` — internal, drives `v3_core`/`compile`; return shapes matched by the sole caller `noenv_forms/3`.
- `env_compiler_options/0` — spec `() -> list()`. persistent_term memoization, returns a list on both branches. Total.

## Summary
The module is small and mostly faithful to its callers' contracts. Three defects: a compile-warning option (`nowarn_nomatch`) threaded through three call levels but never consulted; a `receive` in `spawn/1` that can block forever on an untrappable child death; and a newly-upgraded low-confidence crash — `format_errors/1` `function_clause`s on a binary-keyed error entry (executed evidence below).

## Bugfixes

#### `CSCAN-elixir_erl_compiler-nowarn-nomatch-ignored`
- **Finding ID:** CSCAN-elixir_erl_compiler-nowarn-nomatch-ignored
- **Severity:** Low
- **Confidence:** high
- **Location:** `elixir_erl_compiler.erl:142` (`format_warnings/2`) and all `handle_file_warning/3` clauses (lines 153–168)
- **Class:** Dead / unreachable code (leftover parameter); borderline Documented-behaviour mismatch
- **User-visible outcome:** `format_warnings/2` computes `NoWarnNoMatch = proplists:get_value(nowarn_nomatch, Opts, false)` and passes it as the first argument to every `handle_file_warning/3` call, but every clause of `handle_file_warning/3` binds that argument to `_` and never inspects it. Setting `nowarn_nomatch` therefore has no effect on which warnings are emitted: atom-reason `nomatch` warnings are unconditionally suppressed (line 161) and tuple-reason ones such as `{nomatch, {shadow, …}}` are unconditionally shown (line 166→194), regardless of the flag. A caller who passes `nowarn_nomatch` expecting shadow/no-match warnings to be silenced still sees them.
- **Reproduction:** Invoke a compile that reaches `noenv_forms/3` with `Opts` containing `nowarn_nomatch` on a module producing a `{nomatch,{shadow,…}}` warning; the shadow warning is still reported.
- **Suggested fix:** Either consult `NoWarnNoMatch` where nomatch warnings are handled, or, if suppression is deliberately unconditional, drop the now-dead `proplists:get_value` and the threaded argument to make intent explicit.
- **Risk of fix:** Honouring the flag would silence shadow-clause warnings that are currently always emitted; some callers/tests may rely on seeing them. Removing the dead parameter is behaviour-neutral.
- **Evidence:** read `elixir_erl_compiler.erl:141-168` — `NoWarnNoMatch` bound at 142, all `handle_file_warning/3` heads carry `_` in position 1.

#### `CSCAN-elixir_erl_compiler-spawn-untrappable-death-hang`
- **Finding ID:** CSCAN-elixir_erl_compiler-spawn-untrappable-death-hang
- **Severity:** Low
- **Confidence:** medium
- **Location:** `elixir_erl_compiler.erl:33-40` (`spawn/1` receive)
- **Class:** Infinite loop / hang (unguarded `receive`)
- **User-visible outcome:** The `receive` matches only `{'DOWN', Ref, process, _, {ok, …}}` and `{'DOWN', Ref, process, _, {Kind, Reason, Stack, Diagnostics}}`. The child's `try … catch Kind:Reason:Stack` re-exits every error/exit/throw in the 4-tuple shape, so normal API use always matches. But an untrappable death of the worker — `exit(Pid, kill)`, the emulator killing it on memory pressure, or a `system_limit` abort — yields a `'DOWN'` reason (`killed`, etc.) matching neither clause, and the calling compiler process blocks in `spawn/1` forever. There is no timeout and no catch-all `{'DOWN', Ref, …}` clause.
- **Reproduction:** From another process, `exit(WorkerPid, kill)` while `Fun()` runs (or drive the child out of memory); the enclosing `spawn/1` call never returns.
- **Suggested fix:** Add a fall-through clause `{'DOWN', Ref, process, _, Reason} -> erlang:raise(exit, Reason, [])` so any unmodeled child death propagates instead of hanging.
- **Risk of fix:** None for normal paths; only changes behaviour on abnormal termination, turning a hang into a propagated exit.
- **Evidence:** read `elixir_erl_compiler.erl:19-40` — `spawn_monitor` + two-clause `receive`, no catch-all/timeout.

#### `CSCAN-elixir_erl_compiler-format-errors-binary-file-crash`  *(Upgraded from hedged assumption in first pass)*
- **Finding ID:** CSCAN-elixir_erl_compiler-format-errors-binary-file-crash
- **Severity:** Low
- **Confidence:** low
- **Location:** `elixir_erl_compiler.erl:132-139` (`format_errors/1`)
- **Class:** Crash on stray / unexpected input (missing clause)
- **User-visible outcome:** `format_errors/1` dispatches only on `{File, Each} when is_list(File)` (line 134) and `{Mod, Each} when is_atom(Mod)` (line 137). An error entry whose key is a **binary** file name matches neither guard and raises `function_clause`, converting a compiler error report into an opaque crash of the compile job (the real diagnostics in `Each` are never emitted). This masks whatever error the OTP compiler was trying to report. Reachability depends on the OTP compiler/`v3_core` error-list contract: today those passes key error entries by string filename (a list) or module atom, so the binary path is not known to be produced — hence low confidence. It is a latent, contract-gated crash, not a defect triggerable by ordinary Elixir source.
- **Reproduction (executed, OTP 29):** feeding `format_errors([{<<"lib/foo.ex">>, [{10, erl_lint, undefined}]}])` through the exact two-clause dispatch produced `error:function_clause`; the control case with a list filename `"lib/foo.ex"` handled the entry cleanly. See "Did I verify this".
- **Suggested fix:** add a `{File, Each} when is_binary(File)` clause (or normalise the key with `elixir_utils:characters_to_binary/1` before the `foreach`) so a binary-keyed error entry is formatted rather than crashing.
- **Risk of fix:** none for the current string/atom shapes; only adds coverage for an otherwise-fatal input.
- **Evidence:** read lines 132-139; executed the binary-File and list-File dispatch through Erlang (results above). The crash in this file is confirmed by execution; only the *reachability* (does OTP ever emit a binary key?) remains unverified, which is why this is filed at low confidence rather than left as a ruled-out assertion.

## Risks / caveats
- `noenv_forms/3`, `erl_to_core/2`, and the warning/error formatters depend on the exact result tuples produced by `v3_core:module/2`, `compile:noenv_forms/2`, `sys_coverage:beam_debug_info/1`, and the `{File,[ErrorInfo]}`/`{Line,Module,Desc}` shapes from the Erlang compiler; these are treated as trusted OTP contracts. The binary-`File` variant of that contract is now filed as the low-confidence finding above rather than assumed away.
- `custom_format/2` ends with `Module:format_error(Desc)` (line 206); a warning/error from a module lacking `format_error/1` would crash there, but the Erlang compiler only emits such tuples for modules that export it.

### Ruled out

**Operator / precedence typo**
- Reason: The only guards are `is_list/is_binary` (line 87), `is_atom(Reason)` (line 161), `is_atom(Mod)`/`is_list(File)` (134,137), and `is_binary(Binary)` (line 97); none combine `,`/`;` or `andalso`/`orelse` in a way that could mis-partition.
- Axes examined: mixed opts order for `requires_abstract_format`, atom vs tuple `nomatch` reason, binary vs non-binary compile result | Not examined: none
- Basis: verified

**Missing clause / catch-all** (warning/error dispatch)
- Reason: `handle_file_warning/3` ends with a generic `{Line, Module, Desc}` clause (166) and `custom_format/2` with a generic `Module`/`[]` fallback (202,205); the `try/catch` in `spawn/1` covers all three exception classes. Note: `format_errors/1`'s dispatch is *not* total (no binary-`File` clause) — that gap is now the `format-errors-binary-file-crash` finding, not a ruled-out item.
- Axes examined: unknown warning module, `[]` module, sys_core_fold nomatch atom/tuple, error/exit/throw from `Fun`, binary-keyed error entry (executed → function_clause, promoted to finding) | Not examined: none
- Basis: verified

**Wrong atom / wrong symbol**
- Reason: Atoms used (`beam_debug_info`, `parse_transform`, `nowarn_nomatch`, `nomatch`, `shadow`, `no_effect`, `make_fun`) match the OTP compiler's descriptor vocabulary; `nowarn_nomatch` is correctly named but unused (reported above).
- Axes examined: option atoms in `requires_abstract_format`, descriptor atoms in `custom_format` | Not examined: exact `sys_core_fold`/`v3_core`/`erl_lint` descriptor-tag stability *across OTP versions*
- Basis: asserted
- Why still asserted: the outstanding axis is whether the descriptor tags (`nomatch`, `ignored`, `no_effect`, `shadow`) keep their names in *other/future* OTP releases. That is not executable in this sandbox — I can only observe the tags emitted by the single OTP 29 present, which cannot settle cross-version stability. Reading `sys_core_fold` would tell me the current tags, not which versions disagree, so per the re-review rule the axis stays `Not examined` and the entry stays `asserted`.
- Depends on: OTP `sys_core_fold`/`v3_core`/`erl_lint` — warning descriptor atoms retain their current names across supported OTP versions.

**State / accumulator threading error**
- Reason: `copy_diagnostics/1` merges child `Head` with parent `Tail` once (`Head ++ Tail`, line 47); no recursion or stale-variable reuse; `CodeDiagnostics` reset to `{[], Log}` for the child is intentional.
- Axes examined: `undefined` diagnostics, `{Tail, Log}` present in parent/child | Not examined: none
- Basis: verified

**Off-by-one / wrong arithmetic**
- Reason: No arithmetic on lines/columns/sizes in this file; line numbers are passed through verbatim from OTP descriptors.
- Axes examined: line values in warnings/errors (passed unchanged), `{line,0}` for beam_validator | Not examined: none
- Basis: verified

**Contract violation** (return shape)
- Reason: `noenv_forms/3` returns `{Module, Binary}` only on the `is_binary(Binary)` clause; every other outcome raises via `compile_error`/`exit`, matching its `-spec`.
- Axes examined: 4-tuple ok with non-binary, 3-tuple ok, error, unmatched `_` | Not examined: none
- Basis: verified

**Crash on stray / unexpected input**
- Reason: Public entries guard their argument shapes (`is_list`/`is_binary`) or accept `any()` (`spawn/1`). The one stray-input path that *does* crash — a binary-keyed error entry hitting `format_errors/1` — has been promoted out of this dismissal into the low-confidence finding `format-errors-binary-file-crash` (executed: `function_clause`). Nothing remains ruled out under this class by assertion.
- Axes examined: opts list variants, non-binary compile output, empty error list, **binary-typed `File` in error tuples (executed OTP 29 → function_clause; promoted to finding)** | Not examined: none
- Basis: verified

**Resource leak on error path**
- Reason: `spawn_monitor` yields an auto-cleaned monitor; on both `'DOWN'` clauses the process is already dead and the monitor consumed. No files/ETS/ports opened here.
- Axes examined: ok path, error path | Not examined: untrappable-kill path (no leak, but hangs — reported separately)
- Basis: verified

**Silent corruption**
- Reason: Diagnostics merge preserves both halves; warning/error formatting only produces human messages, not data consumed downstream.
- Axes examined: diagnostics ordering (`Head ++ Tail`), undefined branches | Not examined: none
- Basis: verified

**Infinite loop / hang**
- Reason: Reported as a finding (untrappable child death); the ordinary paths always deliver a matching `'DOWN'`.
- Axes examined: normal exit, caught exception, external kill | Not examined: none
- Basis: verified

**Quadratic-where-linear / Repeated work**
- Reason: `env_compiler_options/0` memoizes via persistent_term; `Head ++ Tail` runs once per spawn, not in a loop; no per-iteration recompilation.
- Axes examined: repeated `env_compiler_options` calls, single diagnostics merge | Not examined: none
- Basis: verified

**Dead / unreachable code**
- Reason: The unused `nowarn_nomatch` threading is the one instance and is reported as a finding.
- Axes examined: `handle_file_warning` arg1 usage | Not examined: none
- Basis: verified

**Documented behaviour mismatch**
- Reason: `nowarn_nomatch` ineffectiveness reported above; other functions match their `-spec`s.
- Axes examined: each export vs spec | Not examined: none
- Basis: verified

**Race condition / shared mutable state**
- Reason: `persistent_term:put(?MODULE, …)` may be executed by concurrent callers but always writes the identical computed option list; last-writer-wins is harmless.
- Axes examined: concurrent first-call to `env_compiler_options/0` | Not examined: none
- Basis: verified

**Wrong scope / order of operations**
- Reason: Diagnostics captured before spawn and merged after join in the correct order; warnings formatted before the success return.
- Axes examined: diagnostics capture/merge order, warning-before-return | Not examined: none
- Basis: verified

**Encoding / unicode mishandling**
- Reason: `File` conversions go through `elixir_utils:characters_to_list/1` (line 88) and `characters_to_binary/1` (135,144). I executed the malformed-input axis: a lone surrogate `<<237,160,128>>` (CESU-8 U+D800) makes `unicode:characters_to_list` return `{error,[],<<237,160,128>>}` and `unicode:characters_to_binary` return `{error,<<>>,<<237,160,128>>}`; `elixir_utils` matches the `{error, Encoded, Rest}` clause (elixir_utils.erl:141/150) and raises a defined `UnicodeConversionError` via `conversion_error/3` (line 154). So malformed/lone-surrogate `File` input yields a deterministic, defined exception at `noenv_forms/3` line 88 — not silent corruption and not an unguarded crash. A trailing invalid byte (`<<"ok",255>>`) likewise returns `{error,"ok",<<255>>}`, same defined path.
- Axes examined: binary File → list Source (executed, valid + lone-surrogate + trailing-invalid-byte), list File → binary in error/warning formatting | Not examined: none
- Basis: verified

**Unclassified**
- Reason: No composite/one-off defect beyond the three reported.
- Axes examined: whole-file read | Not examined: none
- Basis: verified

### Did I verify this?

- Operator/precedence, Missing-clause, State-threading, Off-by-one, Contract-violation, Silent-corruption, Infinite-loop, Quadratic/Repeated-work, Dead-code, Documented-behaviour, Race, Wrong-scope, Unclassified — all `verified`: I read the relevant lines and tried each named axis; `Not examined` reads `none`.
- **Crash on stray input** — now `verified`. I executed the previously-unexamined "binary-typed `File` in error tuples" axis on OTP 29: `format_errors([{<<"lib/foo.ex">>, [{10,erl_lint,undefined}]}])` raised `error:function_clause`, while the list-keyed control formatted cleanly. Because the crash is real but its reachability depends on the OTP compiler ever emitting a binary key, I moved it out of the ruled-out list into the low-confidence finding `format-errors-binary-file-crash` rather than leaving it as an assertion. `Not examined: none`.
- **Encoding/unicode** — now `verified` (was `asserted`). I executed the lone-surrogate/invalid-byte axis through `unicode:characters_to_list`/`characters_to_binary` and confirmed `elixir_utils` turns the `{error,…}` result into a defined `UnicodeConversionError`. `Not examined: none`.
- **Wrong atom / wrong symbol** — remains `asserted`. The outstanding axis is descriptor-tag stability *across OTP versions*, which cannot be executed with a single installed OTP (29); observing this version's tags does not settle other versions, and reading `sys_core_fold` would not clear the axis. It honestly stays `Not examined` / `asserted`, with the depends-on line naming where a cross-version check would run.

### Dropped in re-review

- None. All three prior findings are preserved. The two previously-`asserted` ruled-out entries whose axes I could execute (Crash-on-stray-input, Encoding/unicode) were resolved — one upgraded to a low-confidence finding on executed evidence, one confirmed `verified` — and the one axis that is genuinely not executable here (cross-OTP-version descriptor-tag stability) was left `asserted` as required.
