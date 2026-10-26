# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/parallel_require.ex
<!-- sha256: 0e40278c1cbfd40255cb72286f6ff153fd00562794f76a902bd0ce87ee8fcac1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:28:59Z_

Both behaviors confirmed. I have enough to write the report.

- The binary is Elixir 1.21.0-dev built from this source (stack traces cite `lib/kernel/parallel_require.ex:17`).
- Requiring a file executes its body (by-design code execution; caller supplies the paths).
- The error branch's `exit({:shutdown, 1})` terminates the calling process (the post-call line never printed; process exit status 1).

---

# Security review — `lib/elixir/lib/kernel/parallel_require.ex`

## Summary
This is a 22-line deprecated shim whose public `files/1,2` delegates verbatim to `Kernel.ParallelCompiler.require/2`; it introduces no sink of its own, and no untrusted-data transformation happens inside the file, so there are no security findings.

## Security

`No findings`

The file's entire behavior:
- `files(files, callback)` when the second arg is an arity-1 function → rewrites to `files(files, each_file: callback)` (legacy-callback shim).
- `files(files, options)` when the second arg is a list → calls `Kernel.ParallelCompiler.require(files, options)`, returns `modules` on `:ok`, or `exit({:shutdown, 1})` on `:error`.

Every security-relevant action (reading/compiling/executing the files, parallelism, callback dispatch, path handling) occurs inside `Kernel.ParallelCompiler.require/2`, which is a different file and out of scope for this review. This file neither parses, transforms, joins, expands, nor atomizes any value; it forwards its arguments unchanged.

### Ruled out

- **Code execution via `require` (Code-execution class).** `Kernel.ParallelCompiler.require/2` compiles and runs the bodies of the supplied files — this is the documented, sole purpose of the function, and the file list is passed by the caller, not derived from external input inside this file. Requiring attacker-chosen files is RCE-by-design (equivalent to `Code.require_file`); it is a caller responsibility, not a defect in this wrapper. *Basis: verified* — `elixir -e 'Kernel.ParallelRequire.files(["/tmp/side_effect.exs"])'` executed the file body (`/tmp/PROOF_ran.txt` written; "required file body ran" printed).
- **`exit({:shutdown, 1})` on compile error (Unclassified / availability).** On `{:error, _, _}` the function terminates the calling process instead of returning an error tuple. This is the intended, historical semantics of the deprecated module (the non-deprecated `Kernel.ParallelCompiler.require/2` returns `{:error, …}` for callers wanting graceful handling). It is not an escalation: any file that reaches the error branch was already going to be compiled/executed, which is strictly more dangerous than a clean shutdown. *Basis: verified* — driving `files/2` with an invalid file exited the VM with status 1 and the statement sequenced after the call never ran.
- **Path handling / traversal (File-operations class).** No `Path.join`/`expand`/wildcard or any path manipulation exists here; file paths are forwarded untouched to the compiler. No traversal is introduced by this file. *Basis: verified by full read of the 22-line file.*
- **Callback / behaviour dispatch (Protocol-dispatch class).** The `each_file` callback that gets applied downstream is caller-supplied, not externally derived, and this file only wraps it in a keyword list. *Basis: verified by full read.*
- **Dead defensive code (guards).** Both clause guards (`is_function(callback, 1)`, `is_list(options)`) are reachable through the documented API — the legacy single-callback form and the options-list/default `\\ []` form respectively. Neither guard is a safety predicate; a second argument that is neither routes to `FunctionClauseError`, but reaching that requires the caller to violate the documented `callbacks`-list contract (out of scope). *Basis: verified by full read.*
- **Atom creation, command execution, deserialization, crypto, Unicode, shared mutable state, network.** None of these sink shapes appear anywhere in the file. *Basis: verified by full read of the complete file.*

## Risks / caveats
- The review is correctly file-scoped: the substantive attack surface (parallel compilation, worker processes, file reads, callback execution, resource/DoS behavior under many/large files) lives in `Kernel.ParallelCompiler` and its worker machinery, which this file only calls. Those modules were **not** reviewed here and should be assessed on their own; a clean bill for this shim says nothing about them.
- Absence claims above are quantified over the entire file, which was read in full (22 lines) — a small, self-contained search space — rather than over a partial search.
- Residual, non-security note: consumers migrating off this deprecated module should prefer `Kernel.ParallelCompiler.require/2`, whose `{:error, …}` return replaces the `exit/1` used here; that is an API-robustness consideration, not a vulnerability.

