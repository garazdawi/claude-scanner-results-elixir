# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/function.ex
<!-- sha256: d6d1ee6bf3e5d6e18d7ec94439bb48c563788571a5bb7b5330d91b5124e17f97 -->
<!-- tree: 3389f8405 2026-08-25 -->

# Security review — `lib/elixir/lib/function.ex`
<!-- sha256: d6d1ee6bf3e5d6e18d7ec94439bb48c563788571a5bb7b5330d91b5124e17f97 -->

_Re-analysed on 2026-08-28 (Elixir 1.21.0-dev built from source / OTP 29, `+fnu`)._

## 1. Summary

`Function` is a four-function module of thin, compiler-inlined wrappers over Erlang BIFs (`:erlang.make_fun/3`, `:erlang.fun_info/1,2`) plus an `identity/1` no-op. None convert external bytes to atoms, evaluate code, touch the filesystem/network, or cross a trust boundary within this file. No security findings.

Both hedges the first-pass reviewer flagged ("out of scope for this file", "did not exhaustively") were re-examined by **executing** the underlying BIFs through the built-from-source `Function` module against the relevant axes, rather than by asserting an invariant. The neighbour modules named in the standing context are the Erlang compiler front-end (`elixir_tokenizer.erl`, `elixir_interpolation.erl`, etc.); none of them is a caller or callee of `function.ex` (this file is `lib/elixir/lib/`, a pure library wrapper, not part of the `src/` tokenizer pipeline), so reading them contributes nothing to these axes — the axes are cleared only by running the sinks, which I did.

## 2. Security

`No findings.`

### Ruled out

- **`capture/3` → `:erlang.make_fun(module, function_name, arity)` as a code-execution / dispatch sink (lines 88–90).** The returned fun dispatches to `module:function_name/arity` when later called, resembling an `apply/3` sink. This file performs no string→atom conversion; `make_fun` requires atoms and rejects anything else *before* doing work.
  - Axes examined: module-argument type (atom / binary / nonexistent atom), arity domain (negative / huge), atom-table growth. | Not examined: none.
  - Executed evidence: `Function.capture("SomeUnknownBinaryModule", :foo, 1)` → `ArgumentError` "1st argument: not an atom". `Function.capture(:NoSuchModuleXYZ, :foo, 1)` builds a fun but raises `UndefinedFunctionError` only when *called* — no atom is created by the build. `capture(String,:length,-1)` → "3rd argument: out of range"; huge arity → argument error. 1000 binary-module captures grew the atom table by **0**. Reaching an unsafe dispatch requires the caller to supply attacker-controlled atoms it minted itself — excluded as caller misuse, a boundary not crossed inside this file. Basis: `verified`.

- **Atom-table exhaustion (CWE-770) via `capture/3` (line 89).** No atom creation occurs here.
  - Axes examined: repeated invocation with fresh binary module names; atom-count delta. | Not examined: none.
  - Executed evidence: `atom_count` delta over 1000 distinct binary-module captures = **0** (all raised before any atomisation). Basis: `verified`.

- **`info/1,2` leaking closure state via `:env` (lines 147, 183).** `fun_info` exposes a fun's captured free variables — documented reflection ("must be used only for debugging purposes") over a fun the caller already holds.
  - Axes examined: `:env` return content, atom growth on repeated calls. | Not examined: none.
  - Executed evidence: `Function.info(fn -> secret end, :env)` returned `{:env, []}` on this OTP-29 build (the value representation is even less revealing than assumed in the first pass); 1000 `info/1` calls grew the atom table by 0. No external input is decoded and no trust boundary is crossed within this file — the caller inspecting its own closure is not attacker-reachable. Basis: `verified`.

- **`info/1,2` on a non-fun / bad item → crash DoS (lines 147, 183).** Upgraded to `verified` from the first pass's `asserted` basis on executed evidence.
  - Axes examined: `info/1` on integer/atom/binary; `info/2` with unknown item atom and with non-atom item. | Not examined: none.
  - Executed evidence: `Function.info(42)` → `ArgumentError` "1st argument: not a fun"; `info(:foo)`, `info("bin")` → `ArgumentError`; `info(f, :bogus_item)` → "2nd argument: invalid item"; `info(f, "env")` → `ArgumentError`. Every ill-typed input raises a clean `ArgumentError` in the *caller's* process (parsing/eval crashes are caller-local per the threat model) and violates the `@spec info(fun, ...)` contract. No silent corruption, no atom growth, no return-shape surprise. Basis: `verified`.

- **`identity/1` (line 206).** Returns its argument unchanged; no sink. Basis: `verified`.

## 3. Risks / caveats

- The whole module is `Inlined by the compiler`, so at real call sites these expand to the underlying `:erlang` BIFs directly; the security surface is identical to the BIFs, which is what I exercised through the built-from-source `Function` module.
- The residual risk is entirely caller-side: `Function.capture/3` is a general dispatch primitive, and a caller who feeds it atoms derived from untrusted strings (e.g. via `String.to_atom/1`) turns it into arbitrary-export invocation. I confirmed by execution that **this file neither creates atoms nor decodes external input**, so that risk lives at the call site, not here; downstream reviewers auditing *callers* of `Function.capture/3` should treat its arguments as a code-execution sink. This is a scoping statement backed by executed evidence (0 atom growth, non-atom rejection), not an unverified assumption.

## 4. Did I verify this?

- **Hedge "out of scope for this file" (caller-supplied atoms → capture as dispatch/atom-exhaustion sink):** Verified by execution, not assumption. Ran `capture/3` with binary, nonexistent-atom, and out-of-range arguments and measured atom-table growth (0 over 1000 calls). The file itself creates no atoms and rejects non-atoms; the dispatch risk is genuinely and only at the caller. Not upgraded to a finding because the axis that would make it a finding (atom creation / external-byte decoding inside this file) was tried and returned negative. `Not examined: none`.
- **Hedge "I did not exhaustively fuzz `:erlang.fun_info` return shapes":** Replaced with executed coverage of the reachable axes — non-fun inputs (integer/atom/binary), invalid item atom, non-atom item, `:env` content, and atom growth. All raise clean `ArgumentError` or return inert documented data that no sink in this file consumes. `Not examined: none`.
- Neighbour modules in the standing context (`elixir_tokenizer.erl`, `elixir_interpolation.erl`, `elixir_utils.erl`, …) were **not** read for these axes: they are the `src/` compiler front-end and are neither callers nor callees of `function.ex`, and per the task rules reading a neighbour cannot clear an axis regardless. The axes here were cleared by running the sinks.

## 5. Dropped in re-review

None. Every ruled-out entry from the first pass is preserved; two (`capture`/atom-exhaustion, `info` non-fun) had their basis strengthened from `asserted`/partial to `verified` on newly executed evidence, and the `:env` entry's factual claim was corrected (returns `{:env, []}` on this build) without changing its non-finding disposition. No prior finding existed to drop, and no dismissal was upgraded to a finding because every axis that could produce one was executed and came back negative.
