# Cluster cross-file findings

No cluster cross-file findings

`git.ex` and `path.ex` are two independent, parallel implementations of the `Mix.SCM` behaviour (`scm.ex`); they do not call each other and share no mutable state. I verified that every behaviour callback is implemented by both with the documented return shapes, and that the callers that dispatch across SCMs polymorphically do so safely:

- `Mix.Dep.Fetcher.do_fetch/3` (`dep/fetcher.ex:52-89`) routes non-fetchable deps (Path, `fetchable? == false`) into the branch that never calls `update/1` or `checkout/1`, so `Mix.SCM.Path.checkout/1`'s unconditional `Mix.raise` (`path.ex:78-81`) is never reached on the fetch path; Git's `fetchable? == true` (`git.ex:10-12`) is the only one that reaches `checkout/update`.
- `Mix.Dep.check_lock/1` (`dep.ex:445-465`) handles the full `:ok | :mismatch | :outdated` range that `Mix.SCM.Git.lock_status/1` (`git.ex:72-92`) can emit, and `Mix.SCM.Path.lock_status/1` (`path.ex:62-65`) only ever returns `:ok`.
- `Mix.Dep.Converger.converge?/2` (`dep/converger.ex:378-384`) guards `scm1 == scm2` before invoking `scm1.equal?/2`, honoring the behaviour's "options guaranteed to belong to the same SCM" precondition (`scm.ex:116-121`), so Git's `equal?` (`git.ex:95-97`) and Path's `equal?` (`path.ex:67-70`) are never cross-applied.
- The opts keys each SCM populates in `accepts_options/2` (Git sets `:checkout` and `:dest`, `git.ex:41-62`; Path sets `:dest`, `path.ex:24-55`) are each read back only by that same SCM's callbacks or by shared callers using `:dest`, which both set.

No type/arity mismatch, resource-pairing error, state-machine inconsistency, or broken inherited assumption spans the two files.
