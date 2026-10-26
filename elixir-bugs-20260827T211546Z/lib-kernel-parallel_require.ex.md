# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/parallel_require.ex
<!-- sha256: 0e40278c1cbfd40255cb72286f6ff153fd00562794f76a902bd0ce87ee8fcac1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:29:40Z_

## Summary

`Kernel.ParallelRequire` is a small deprecated shim that delegates to `Kernel.ParallelCompiler.require/2`, and its two-clause dispatch and result matching are correct and total.

## Bugfixes

No findings

The module has exactly one public function, `files/2` (`parallel_require.ex:10-21`), both clauses of which behave correctly:

- The function-callback clause (`:12`) wraps an arity-1 function as `each_file: callback`, which matches the `:each_file` option documented for `Kernel.ParallelCompiler.require/2` (`parallel_compiler.ex:262-263`) — the historical contract of this shim.
- The options-list clause (`:16`) matches the delegate's full documented return surface: `require/2`'s `@spec` (`parallel_compiler.ex:278-280`) returns exactly `{:ok, [atom], _}` or `{:error, _, _}`, both of which are handled (`:18-19`), so no `CaseClauseError` is reachable on any spec-conforming return.

## Risks / caveats

- Verification here is by inspection plus reading the delegate's `@spec` and `@doc` (`parallel_compiler.ex:278-283`), not by executing the deprecated path; I did not run a deprecation-warning-driving script, but the control flow is small enough to settle statically.
- Passing a function of arity ≠ 1 as the second argument falls through both guarded clauses and raises `FunctionClauseError`, but that violates the documented callback contract (arity-1 `each_file`) and so is out of scope per the "ignore misuse of the documented `@spec`/contract" rule — not reported as a finding.
- Behavior on a non-list, non-function second argument is likewise a contract violation and intentionally out of scope.

### Ruled out
- **Result-shape mismatch causing `CaseClauseError`** — the `case` on `require/2`'s return handles both `{:ok, _, _}` and `{:error, _, _}`, which is the complete set the delegate's `@spec` promises (`parallel_compiler.ex:278-280`). No third shape exists to fall through. Basis: asserted (read against the delegate's authoritative `@spec`, the mechanism that fixes the return contract).
- **Wrong callback option (`each_file` vs `each_module`)** — the arity-1 function is mapped to `each_file`, matching the documented option that passes the file to the callback (`parallel_compiler.ex:262-263`) and the historical semantics of this shim. Basis: asserted (verified against the delegate's documented option list).

