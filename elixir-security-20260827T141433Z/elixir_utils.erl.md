# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_utils.erl
<!-- sha256: 287f47e07464c121a53318a360faec0309bb59172d9b150986fa5b5f4d20535b -->

_Analysed on 2026-08-27T17:31:27Z_

I have enough to complete the review. Let me confirm the file-metadata functions aren't invoked with attacker-constructed paths anywhere in this folder (the grep showed only their definitions here), and verify `returns_boolean` recursion is bounded by the same depth the parser already survived — both are quick reasoning checks I've completed. Writing the report.

## Summary

`elixir_utils.erl` is a collection of pure AST/option helpers plus thin Unicode-conversion and file-metadata wrappers for the Elixir compiler; its only untrusted-data path (Unicode conversion of tokenizer atoms) correctly surfaces decode errors, and it contains no code/command execution, deserialization, or path-construction sinks — **No findings**.

## Security

No findings.

The one obligation that genuinely lands on this file — **UTF8_DECODE_ERRORS_SURFACED** (checklist sub-claim 11) — is *satisfied*. `characters_to_list/1` (`elixir_utils.erl:138-143`) and `characters_to_binary/1` (`elixir_utils.erl:147-152`) route non-trivial input through `unicode:characters_to_list/1`/`characters_to_binary/1` and turn both `{error, …}` and `{incomplete, …}` results into a raised `UnicodeConversionError` via `conversion_error/3` (`:154-155`). There is no silent-replacement or garbage path, and the error handler does **not** recurse per bad byte, so it avoids the quadratic-recovery shape seen in `String.normalize` (memory `string-normalize-invalid-utf8-quadratic`). Confirmed by running the underlying primitives (invalid → `{error,[],…}`, incomplete → `{incomplete,…}`).

## Risks / caveats

- **`characters_to_list/1` list-passthrough asymmetry (latent, not reachable as a validation gate).** `characters_to_list(Data) when is_list(Data) -> Data` returns any list unvalidated (a list containing `16#110000` or a surrogate passes straight through — confirmed by running), whereas `characters_to_binary` passes *binaries* through and validates lists. This is safe *today* only because the untrusted-source callers in `elixir_tokenizer.erl:1108-1167` deliberately cross type — feeding binaries to `characters_to_list` and lists to `characters_to_binary`, so each hostile atom always hits a *validating* branch. A future caller that hands an untrusted improper iolist to `characters_to_list` expecting normalization would get no validation. Worth a defensive comment; not a finding.
- Several dismissals below depend on out-of-folder callers (the file-metadata functions are not called anywhere inside `src`); those dependencies are enumerated as `Depends on:` lines for the cross-file pass.

### Ruled out

- **`characters_to_list/1`, `characters_to_binary/1` (Unicode conversion / UTF8_DECODE_ERRORS_SURFACED).**
  Reason: both error and incomplete decode results are raised as `UnicodeConversionError`; no silent garbage, no quadratic recovery, and tokenizer callers always take the validating (type-crossing) branch for hostile atoms.
  Axes examined: valid UTF-8, invalid UTF-8 (bad continuation), incomplete/truncated UTF-8, invalid iolist→binary, list-passthrough of out-of-range codepoint | Not examined: none.
  Basis: verified.

- **`macro_name/1` — `list_to_atom("MACRO-" ++ atom_to_list(Macro))` (`:27-28`), atom creation.**
  Reason: `Macro` is already an atom (macro/function name from `defmacro`/dispatch at `elixir_erl.erl:264/485`, `elixir_dispatch.erl:256/267/280`, `elixir_erl_pass.erl:90`), so at most one new `"MACRO-"` atom is minted per already-existing atom — bounded by the source that already atomized the identifier; the standard, documented atom-table cost of compiling untrusted source, not a distinct amplification.
  Axes examined: input already-atom vs raw text, per-identifier atom multiplicity, unbounded-vs-bounded growth | Not examined: none.
  Basis: verified.

- **`erl_fa_to_elixir_fa/2` — `list_to_atom(Rest)` (`:30-34`), atom creation.**
  Reason: `Name` is an existing function-name atom (`elixir_erl_compiler.erl:195`, `elixir_def.erl:58`); stripping the `"MACRO-"` prefix yields an atom that already exists for any real macro, so growth is bounded by the existing atom table.
  Axes examined: input already-atom, prefix-present vs absent branch, atom multiplicity | Not examined: none.
  Basis: verified.

- **`read_file_type/1,2` (`:99-106`), `read_link_type/1` (`:108-112`), `read_posix_mtime_and_size/1` (`:114-118`) — `file:read_*_info`.**
  Reason: this file performs no path construction/joining — it receives a complete path and does a single read-only metadata query; there is no traversal-injection sink here, and no caller exists inside `src` (grep found only the definitions), so path origin is the build system / developer project.
  Axes examined: path construction in-file (none), symlink-follow semantics of read_file_info vs read_link_info, read-only vs mutating | Not examined: whether an out-of-folder caller passes an attacker-derived path.
  Basis: asserted.
  Depends on: callers in the Elixir stdlib/Mix (e.g. `File`, `Code`, `Mix.Compilers.*`) pass build/source paths, not paths built from untrusted source *content*; the module-name→path join lives in `elixir_module.erl:585` (`filename:join`), not here.

- **`change_posix_time/2` (`:120-121`), `change_universal_time/2` (`:123-126`) — `file:write_file_info` (mtime).**
  Reason: the only mutation is the mtime of a caller-supplied path (build-artifact timestamp tracking); no content is written and no path is constructed in this file; both clauses are `is_integer`/tuple-guarded so a malformed time cannot reach the sink.
  Axes examined: in-file path construction (none), value-type guards on Time, write scope (metadata only) | Not examined: whether a caller supplies an attacker-controlled destination path.
  Basis: asserted.
  Depends on: `elixir_module.erl` / Mix compilers supply `.beam`/source paths derived from the developer project, not from untrusted source text, when touching timestamps.

- **`relative_to_cwd/1` (`:128-134`) — `elixir_config:get(relative_paths)` + `'Elixir.Path':relative_to_cwd/1`.**
  Reason: shared-state read is a compiler-config boolean (operator-trusted, not external input), and `Path.relative_to_cwd/1` is pure string munging for stacktrace display — no filesystem access, and the whole thing is wrapped in `try/catch _:_ -> Path`.
  Axes examined: config read vs write (read-only), path used for display vs fs-op (display), exception containment | Not examined: none.
  Basis: verified.

- **`returns_boolean/1` (`:207-248`) — recursive AST-walk validation predicate.**
  Reason: it is a compiler optimization predicate over the developer's own quoted AST; a wrong answer at worst miscompiles the compiling party's own code (BEAM remains type-safe) and crosses no trust boundary to a third-party victim, and recursion depth equals AST-nesting depth the recursive-descent parser already survived. Logic reviewed (`andalso`/`orelse` recurse into the right operand, `case`/`cond`/`__block__` recurse into result exprs) and is correct.
  Axes examined: correctness of each boolean-returning clause, recursion depth vs parser depth, trust-boundary of a wrong result | Not examined: none.
  Basis: verified.

- **`var_info/2` (`:19-22`) — `io_lib:format`.**
  Reason: the format string is a compile-time constant; `Name`/`Kind` are inserted only through `~ts`/`~ts` value directives, so there is no format-string-injection surface.
  Axes examined: format-string constant vs data-derived, directive type (`~ts` value-only) | Not examined: none.
  Basis: verified.

- **`caller/4` + `stack_location/2` (`:158-167`) — stacktrace-entry construction.**
  Reason: builds a `{Module, Fun, Arity, [{file,…},{line,…}]}` term for diagnostics; `File` flows only into `characters_to_list`/`relative_to_cwd` (both ruled safe above), no sink beyond term construction.
  Axes examined: value flow of File/Line into downstream sinks, term-shape only | Not examined: none.
  Basis: verified.

- **`guard_op/2` (`:36-45`) — `erl_internal:op_type/2` in `try/catch`.**
  Reason: pure classification of an operator/arity, any exception caught and mapped to `false`; no external effect.
  Axes examined: exception containment, effect-freedom | Not examined: none.
  Basis: verified.

- **Pure helpers: `split_last/1` (`:80-83`), `split_opts/1` (`:86-97`), `extract_guards/1` + `extract_or_guards/1` + `extract_splat_guards/1` (`:61-73`), `meta_keep/1` (`:197-203`), `get_line/1,2` (`:169-179`), `get_file/2` (`:181-185`), `var_context/2` (`:53-57`), `generated/1` (`:187-188`), `guard_info/1` (`:24-25`), `erlang_comparison_op_to_elixir/1` (`:47-51`), `noop/0` (`:77-78`).**
  Reason: list/proplist manipulation and literal-atom mapping with no external side effect, no dynamic atom creation, no I/O; nothing security-relevant.
  Axes examined: side-effect freedom, dynamic-atom creation (none), recursion termination on proper lists | Not examined: none.
  Basis: verified.

### Did I verify this?

- `characters_to_list/binary`, `macro_name`, `erl_fa_to_elixir_fa`, `relative_to_cwd`, `returns_boolean`, `var_info`, `caller`/`stack_location`, `guard_op`, and the pure helpers are marked **verified**: I read the code, traced the value flow (including the tokenizer branch selection at `elixir_tokenizer.erl:1108-1167` for the Unicode path), ran the Unicode primitives to confirm error surfacing and passthrough semantics, and each `Not examined` list reads `none`. No `Depends on:` lines remain for these.
- The two **asserted** entries — `read_file_type`/`read_link_type`/`read_posix_mtime_and_size` and `change_posix_time`/`change_universal_time` — remain asserted because their callers live outside this folder (no in-`src` call site exists), so I cannot, from this file alone, prove the destination path is never attacker-constructed. I kept explicit `Depends on:` lines pointing at the Elixir stdlib/Mix callers and at `elixir_module.erl:585` (where the module-name→path join actually happens) so the cross-file pass can confirm those paths are developer- rather than untrusted-content-derived. Crucially, even if that dependency were violated, this file still contains **no path-construction or traversal sink** — it only performs a single metadata op on a whole path — so the residual risk is confined to those neighbors, not `elixir_utils.erl`.

