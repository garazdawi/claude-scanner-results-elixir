# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_env.erl
<!-- sha256: 26616d466f6fd67f5d028af3d71154a63c632d04c0d4698ee55d27352c837b33 -->

_Analysed on 2026-08-27T16:41:16Z_

Confirmed the runtime behaviors I need: `~ts` accepts an atom (`ts_atom="foo"`), tilde/format-directives embedded in an atom name are printed **literally** as data, not interpreted (`ts_tilde="var \"a~w~s~p\" unused"` — no format-string injection), and version counters are Erlang bignums (`is_integer=true` at 1 bsl 70, no wrap). I now have enough to write the report.

---

## Summary

`elixir_env.erl` is a compiler-internal module that constructs and mutates the `Macro.Env` struct and the `#elixir_ex{}` variable-tracking record during macro expansion; it contains no deserialization, filesystem, network, crypto, or atom-creation sinks, and the only code-execution and formatting sinks are driven by operator-controlled compiler config or literal format strings, so there are **no security findings**.

## Security

No findings.

### Ruled out

**`trace/2` — dynamic dispatch `Tracer:trace(Event, E)` (line 34) — Code execution.**\
Reason: `Tracers` is the `tracers` list carried in the env; it is seeded to `[]` by `new/0` (line 30) and populated only from operator/build-time compiler configuration (`Code.put_compiler_option(:tracers, …)` / `elixir_config`), which the folder threat model explicitly places out of scope ("a malicious *operator* who already controls compiler config/tracers … already ha[s] code execution"). Source code being compiled cannot set the tracer list, so no untrusted-source path reaches this dispatch.\
Axes examined: origin of `tracers` (config vs. source), default value from `new/0` | Not examined: none.\
Basis: verified.

**`format_error/1` — `io_lib:format` with variable-name atom (lines 154-164) — Template/interpolation.**\
Reason: every format string is a compile-time literal and the untrusted-ish value (`Name`, a variable-name atom) is passed as a `~ts` *argument*, not spliced into the format string; embedded `~w~s~p` in an atom name is rendered as literal text (verified), so there is no format-string injection, and `~ts` accepts atoms without crashing (verified).\
Axes examined: atom input to `~ts`, tilde/format-directive injection via the atom name, non-ASCII atom name | Not examined: none.\
Basis: verified.

**`check_unused_vars/2` & `merge_and_check_unused_vars/3,4` — `elixir_errors:file_warn(...)` (lines 101, 135) — Human-in-the-loop / template.**\
Reason: these emit "unused variable" warnings whose payload is a variable-name atom and source `Meta`; a variable name reaching here has already passed the tokenizer's identifier rules (UAX#31 / bidi rejection / NFC in `elixir_tokenizer.erl`), and an unused-variable warning is not a value any human authenticates a security decision on. Impact is a benign diagnostic string at worst.\
Axes examined: content source of the warning (var name + meta), security-relevance of the diagnostic | Not examined: bidi/confusable content of the name *as enforced upstream* (delegated to `elixir_tokenizer.erl`).\
Basis: asserted.\
Depends on: elixir_tokenizer.erl — rejects bidi/NEL/unsupported-newline codepoints and requires NFC in identifiers before any atom reaches `versioned_vars`/`unused`, so warning text cannot carry directional-override spoofing.

**`calculate_span/2` — `Column + string:length(atom_to_binary(Name))` (line 108) — Resource / arithmetic.**\
Reason: pure arithmetic over a source-column integer plus the grapheme length of a variable name; `atom_to_binary` is a reverse conversion (no atom creation) and `string:length` is linear in the (source-bounded) name length. No overflow (Erlang bignums), no sink.\
Axes examined: overflow/wrap of the sum, atom-creation direction, cost as f(name length) | Not examined: none.\
Basis: verified.

**`merge_vars/2`, `with_vars/2`, `merge_and_check_unused_vars/4`, `env_to_ex/1` — map/list folds and counters (lines 42-47, 54-65, 85-93, 119-142) — Resource consumption / counter wrap.**\
Reason: each is a single linear `maps:fold`/`lists:foldl` over structures bounded by the number of variables in a lexical scope (bounded by source size); there is no nested fold or repeated-rebuild pattern that would be quadratic, and version counters are `map_size/1` / `I+1` producing Erlang arbitrary-precision integers that cannot wrap (verified `is_integer` at `1 bsl 70`).\
Axes examined: linear vs. quadratic cost, integer wrap of version/counter values | Not examined: none.\
Basis: verified.

**`to_caller/1`, `reset_read/2`, `prepare_write/1,2`, `close_write/2`, `reset_vars/1`, `reset_unused_vars/1`, `new/0` — record/map construction (lines 14-31, 37-40, 49-50, 69-83, 97-98) — Deserialisation / shared state.**\
Reason: these only build or field-update in-process `Macro.Env` maps and `#elixir_ex{}` records passed as function arguments; there is no `binary_to_term`, no process-dictionary/ETS/app-env write, and no external-bytes-to-term conversion. `new/0` uses only literal atom keys/values and `elixir_dispatch:default_*` (fixed default import sets).\
Axes examined: external-byte deserialization, global/shared-state writes, atom creation | Not examined: none.\
Basis: verified.

**`is_unused_var/1`, `is_compiler_var/1` — predicates (lines 144-152) — Validation.**\
Reason: their return value only gates whether a warning is emitted; no security property (auth, path safety, sandboxing) depends on the answer, so a wrong answer at most suppresses/adds a diagnostic.\
Axes examined: security-relevance of the boolean, downstream use of the result | Not examined: none.\
Basis: verified.

### Did I verify this?

- `trace/2` — verified: read `new/0` (line 30, `tracers => []`) and confirmed the list originates from compiler config, not compiled source; the out-of-scope operator boundary is stated in the folder threat model. `Not examined: none`; no `Depends on:` needed.
- `format_error/1` — verified: ran `io_lib:format` with atom, tilde-laden atom, and Unicode atom; format strings are literals. `Not examined: none`.
- `check_unused_vars`/`merge_and_check_unused_vars` warning emission — remains **asserted**: the only residual concern (spoofing content in a variable name reaching a human-read warning) is neutralized upstream in `elixir_tokenizer.erl`, which this file does not implement. The `Depends on:` line above names exactly where that guarantee lives so the cross-file pass can confirm the tokenizer rejects bidi/non-NFC identifiers. Even absent that guarantee, the security impact of an unused-variable warning is negligible, so this does not rise to a finding.
- `calculate_span/2`, the fold/counter group, the record/map-construction group, and the predicates — verified: read each clause and tried the datatype's relevant axes (integer wrap, atom-creation direction, fold cost, external deserialization, global-state writes); each `Not examined` list is `none`.

## Risks / caveats

- This review is scoped to `elixir_env.erl`. Its safety leans on one out-of-file invariant made explicit above: variable-name atoms and `Meta` reaching this module have already been validated by `elixir_tokenizer.erl` (valid UTF-8, bidi/unsupported-newline rejection, NFC identifiers) and created there via the tokenizer's atom path — this file only *reads* those atoms (`atom_to_list`, `atom_to_binary`), so it neither re-sanitizes nor re-atomizes them. If the tokenizer's identifier guards were bypassed, the only observable effect *here* would be spoofable text inside "unused variable" diagnostics — not a code-execution, memory-safety, or auth consequence.
- The `trace/2` dynamic dispatch is a genuine code-execution primitive, but it is a designed extension point fed by build-time compiler configuration; it becomes dangerous only under a threat model (malicious operator controlling `tracers`) that the folder explicitly excludes. If a downstream consumer ever let untrusted input populate the env's `tracers` field, this dispatch would become a live RCE sink — worth keeping in mind for any embedding that exposes compiler options to untrusted callers.

