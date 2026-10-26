# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/behaviour.ex
<!-- sha256: db9cdd62adc2d2a58dd9dd66ddf46e854836df424fd3b29f7ce8f3e6f46af512 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/behaviour.ex
<!-- sha256: db9cdd62adc2d2a58dd9dd66ddf46e854836df424fd3b29f7ce8f3e6f46af512 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (second pass; hedge phrases from first pass resolved with executed evidence)_

## Summary

`Behaviour` is a small, deprecated compile-time macro module (`defcallback/1`, `defmacrocallback/1`, `use Behaviour`) that rewrites developer-written AST into `@callback`/`@macrocallback` attributes. Its only input is the AST the developer writes in their own source at their own compile time; no path carries attacker-controlled data to a dangerous sink. **No security findings.**

This pass targeted the two hedges the first-pass reviewer flagged ("did not exhaustively"):
1. *"did not exhaustively fuzz every malformed AST shape into `split_spec`/`Macro.decompose_call`"* — **resolved by execution.** I pushed five representative shapes (integer literal, string literal, default-arg, remote-call spec, guard clause) through a real compile. Each either raised `ArgumentError` from this module's own gates, or produced well-formed callback AST that `Kernel.Typespec` independently rejected (`Kernel.TypespecError`). No shape reached a sink or produced silent bad output. See "Did I verify this".
2. *"[Kernel.Typespec / Code.fetch_docs] treated as trusted collaborators, not audited"* — **resolved.** I read `Code.fetch_docs/1` (code.ex:2305-2342): its argument here is always the fixed self-reference `__MODULE__`, and it returns `{:error, :module_not_found}`/`{:error, :chunk_not_found}` on missing docs — no attacker-controlled module name, and the only failure mode is the self-inflicted `MatchError` already catalogued below. `Kernel.Typespec` receives only well-formed callback AST derived from the developer's own spec and independently validates it (observed `TypespecError`).

This file is not one of the Unicode-handling files in the standing checklist (tokenizer/interpolation/utils); none of the BIDI / NFC / mixed-script / escape obligations apply to it. It handles no raw source bytes.

## Security

`No findings`

The public API is three compile-time constructs — `defcallback/1`, `defmacrocallback/1`, `__using__/1`. Their input is the AST in the developer's *own* module at *own* compile time. There is no `Code.eval*`/`Code.compile*`, no `String.to_atom`/`binary_to_atom` (the callback name arrives as an atom from `Macro.decompose_call/1`), no `System.cmd`/shell, no `File`/`Path`, no deserialisation, network, crypto, shared mutable state, or interpolation reaching a downstream interpreter (the only `#{}` builds an `ArgumentError` string from an internal `kind` atom and `Macro.to_string/1`). Any "abuse" requires the attacker to control the source being compiled, which already grants compile-time code execution independent of this module — so it crosses no trust boundary this module owns.

### Ruled out

- **`Code.fetch_docs(__MODULE__)` MatchError in injected `__behaviour__(:docs)` (behaviour.ex:111)** — *Not security.* `verified`. Reproduced with `docs: false`: `NoDocs.__behaviour__(:docs)` → `docs raised: MatchError`, while `__behaviour__(:callbacks)` still works (`[foo: 1]`). Confirmed via `code.ex:2307-2336` that `fetch_docs/1` returns `{:error, :module_not_found}`/`{:error, :chunk_not_found}` on a missing docs chunk, failing the `{:docs_v1, …}` match. The argument is the fixed `__MODULE__`; inputs are the static atom `:docs` and the module's own docs-chunk presence. Self-inflicted robustness gap on a deprecated API when compiled `--no-docs`/stripped; no confidentiality/integrity/availability impact beyond the caller crashing their own call.
- **`unquote(name)/unquote_splicing(args)/unquote(return)/unquote(guards)` in `do_callback` (behaviour.ex:78-86)** — *Not security.* `verified`. Ordinary macro hygiene: the unquoted fragments are the developer's own spec AST expanded during their own compilation. Reproduced: valid input `defcallback foo(x :: integer) :: atom` yields `[foo: 1]`; the generated spec is handed to `Kernel.Typespec`, which independently validates it (a bad type such as `foo(x) :: atom when is_integer(x)` raises `Kernel.TypespecError`, not undefined behaviour here).
- **Error-message interpolation `"invalid syntax in #{kind}callback #{Macro.to_string(spec)}"` (behaviour.ex:60)** — *Not security.* `verified`. `kind` is an internal `:def`/`:defmacro` atom; the string is raised as `ArgumentError` and reaches no shell/query/template/interpreter. Reproduced: `defcallback 123 :: atom` and `defcallback Foo.bar(x) :: atom` both raise `ArgumentError` through this branch.
- **`ensure_not_default/1` and the `do_defcallback` `_ -> raise` fallback (behaviour.ex:59-60, 89-93)** — *Not dead code.* `verified`. Both fire: `defcallback foo(x \\ 1) :: atom` → `ArgumentError`; `defcallback Foo.bar(x) :: atom` → `ArgumentError`. They validate developer syntax, not a "this is safe" security predicate.
- **`Code.fetch_docs` / `behaviour_info` / `__behaviour__` dispatch (behaviour.ex:106-111)** — *Not security.* `verified`. All dispatch on `__MODULE__` (the module doing `use Behaviour`); `code.ex:2307` confirms `fetch_docs/1` is called with that fixed atom. No externally-influenced module name, no dynamic `apply/3`.

## Risks / caveats

- **Scope is this file only.** `defcallback`/`defmacrocallback` hand generated callback AST to `Kernel.Typespec`, and `__behaviour__(:docs)` calls `Code.fetch_docs/1`. I read `Code.fetch_docs/1` (code.ex:2305-2342) and confirmed it is called only with the fixed `__MODULE__` and cannot receive attacker-controlled input from this file. `Kernel.Typespec` was exercised behaviourally (it rejects malformed generated specs with `TypespecError`) but its internals were not audited; that module's own review is out of scope. The AST it receives from here is always well-formed callback-spec AST or was rejected earlier by this module's gates.
- **Malformed-AST exhaustiveness (upgraded from hedged assumption in first pass → now verified, not a finding).** The first pass hedged that it "did not exhaustively fuzz." Executed evidence across five representative shapes shows the branch space is closed: a shape is either (a) rejected by `Macro.decompose_call/1` → `ArgumentError` (integer/string/remote-call), (b) rejected by `ensure_not_default/1` → `ArgumentError` (default arg), or (c) passed as well-formed AST to `Kernel.Typespec`, which validates it (`TypespecError`). None of the three outcomes is a security sink. This is not full input-space enumeration, but the three exit paths are structurally exhaustive for the `case`/`fun` control flow in `do_defcallback`/`do_callback`, so no residual security risk remains on this axis.
- Residual risk is limited to the deprecated `__behaviour__(:docs)` `MatchError` (robustness, not vulnerability).

## Did I verify this

Executed against the sandbox toolchain (Elixir 1.21.0-dev, Erlang/OTP 29, `ELIXIR_ERL_OPTIONS=+fnu`):

- **Good path:** `defcallback foo(x :: integer) :: atom` → `behaviour_info(:callbacks)` = `[foo: 1]`. ✓
- **Malformed-AST sweep** (resolves hedge #1), each compiled for real:
  - `defcallback 123 :: atom` → `ArgumentError`
  - `defcallback ("s") :: atom` → `ArgumentError`
  - `defcallback foo(x \\ 1) :: atom` → `ArgumentError` (default-arg gate)
  - `defcallback Foo.bar(x) :: atom` → `ArgumentError` (decompose_call fallback)
  - `defcallback foo(x) :: atom when is_integer(x)` → `Kernel.TypespecError` (well-formed AST, rejected downstream)
- **Docs MatchError** (resolves ruled-out #1): compiled with `Code.compiler_options(docs: false)`; `__behaviour__(:docs)` → `MatchError`, `__behaviour__(:callbacks)` → `[foo: 1]`.
- **Neighbour read** (resolves hedge #2): `Code.fetch_docs/1` at code.ex:2305-2342 — argument is fixed `__MODULE__`; returns `{:error, :module_not_found}`/`{:error, :chunk_not_found}` when docs are absent.

Every prior ruled-out entry is now backed by executed evidence rather than assertion; no axis remains merely asserted, so all are marked `verified`.

## Dropped in re-review

None. Every conclusion from the first pass is preserved; the two hedged caveats were upgraded to executed-verification status (both confirming the original "not security" verdict), and no dismissal was moved to a finding because no plausible sink exists in this file.
