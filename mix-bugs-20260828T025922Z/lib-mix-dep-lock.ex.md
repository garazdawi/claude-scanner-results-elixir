# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/lock.ex
<!-- sha256: 8681dfa9190e722a7a01b01abff2e80c31e775e45d091addb85d720ce937ce8e -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/lock.ex
<!-- sha256: 8681dfa9190e722a7a01b01abff2e80c31e775e45d091addb85d720ce937ce8e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`elixir/lib/mix/lib/mix/dep/lock.ex` is a small two-function lockfile reader/writer. `read/1` behaves correctly. `write/2` ignores its own `:file` option when deciding whether to write, comparing the incoming map against the *default* project lockfile (`read()` with no argument) instead of the target file (`read(lockfile)`). The defect is confirmed by execution; the "which callers hit it" hedge from the first pass is now resolved by exhaustively reading every in-tree caller.

## Bugfixes

#### `CSCAN-lock-write-ignores-file-opt`

- **Finding ID** — `CSCAN-lock-write-ignores-file-opt`
- **Severity** — Medium (downgraded from High: no in-tree caller triggers the divergent path — see caller analysis; the defect is a real API-contract bug reachable via the documented `:file` option and exercised directly by the test suite, but every production call site resolves `:file` to the default)
- **Confidence** — high
- **Location** — `lib/mix/dep/lock.ex:45`, `write/2`
- **Class** — State / accumulator threading (wrong value read for comparison)
- **Impact** — When a caller passes the documented `file:` option (`@type write_opts` lists `file: Path.t()`) pointing at a lockfile other than the project's default, `write/2` compares the new map against the *default* lockfile rather than the target file. Two concrete failures:
  - If the default lockfile happens to equal the new map, the target file is **silently not written**, leaving a stale target lockfile (lost update).
  - Symmetrically, `check_locked: true` fails to raise when the *target* file is out of date (and can spuriously raise / spuriously rewrite when the target already matches but the default differs).
- **Evidence** — `verified` (executed in first pass, unchanged):
  - Run 1: default `mix.lock` = `%{foo: :bar}`, `custom.lock` = `%{foo: :OLD}`, then `write(%{foo: :bar}, file: "custom.lock")` → `custom.lock` remained `%{foo: :OLD}`; the write was skipped because the comparison read the default file.
  - Run 2: same setup with `check_locked: true` → did not raise, though `custom.lock` was actually out of date.
- **Suggested fix** — Change line 45 from `if map != read() do` to `if map != read(lockfile) do`, so the comparison reads the same file that will be written.

## Risks / caveats (upgraded from hedged assumption in first pass)

The first-pass hedge — *"I did not exhaustively confirm the exact behavior of every real caller"* — is now resolved. I enumerated all in-tree `Mix.Dep.Lock.write` call sites via grep and read each:

- **`lib/mix.ex:1007`** — `write(Map.merge(...), file: lockfile)` where `lockfile = Path.join(install_project_dir, "mix.lock")`. This runs inside `File.cd!(install_project_dir, fn -> … end)` (mix.ex:986), and the pushed install-project config sets `lockfile: "mix.lock"` (relative; mix.ex:1127 `install_project_config/1`). So `read()`'s default (`Mix.Project.config()[:lockfile]` = `"mix.lock"`, resolved against cwd = `install_project_dir`) and the explicit `:file` (absolute `install_project_dir/mix.lock`) are the **same file**. No divergence. **Verified** by reading mix.ex:955–1010 and 1122–1134.
- **`lib/mix/dep/fetcher.ex:120`** — `write(lock, opts)` forwarding the fetch opts from `deps.get`/`deps.update`. Those opts (`deps.get.ex:39–43`, `deps.update.ex:65–69`) carry `env`/`target`/`check_locked` but **no `:file`** key, so `lockfile = lockfile()` (default) and `read()` reads that same default. No divergence. **Verified** by reading fetcher.ex:98–120, deps.get.ex, deps.update.ex.
- **`lib/mix/tasks/deps.unlock.ex:42,106`** — `write(%{})` / `write()` with **no opts**. Default == default. No divergence. **Verified.**

Conclusion: the divergent branch is reached only by the direct test-suite calls (`mix_test.exs:263,293,311` pass `file:` to a fixture lockfile) and by any out-of-tree caller using `write_opts`'s `:file`. No production call site currently triggers the lost-update behaviour, which is why severity is Medium rather than High. The code defect is nonetheless real and should be fixed, since the `:file` option is public (`@type write_opts`) and its documented semantics are silently wrong.

`read/1`'s deliberate `else _ -> %{}` swallow and its `Code.eval_quoted` of the lockfile are by-design per the module's threat model (untrusted-project-is-code), not findings.

### Ruled out

- **`with` bare-expression clause in `read/1` silently dropping the merge-conflict check** — `with` permits non-`<-` expressions; `assert_no_merge_conflicts_in_lockfile/2` runs as a side-effecting statement, and its `Mix.raise` propagates out of the `with` (it is not a matchable failure, so it is not routed to the `else`). The conflict guard works. Not a bug.
  - Axes examined: control-flow semantics of a bare expression inside `with`; raise-vs-else routing. | Not examined: none — this is a language-level compile/eval fact confirmed by reading; there is no input axis that changes whether a raised exception is caught by `else` (only `<-` mismatches route to `else`).
- **`read/1` returning `%{}` on parse/eval failure** — matches `@spec read :: map()` and the documented swallow-and-default design. Intentional, not a defect.
  - Axes examined: return-shape contract vs `@spec`; documented intent. | Not examined: none — the `%{}` fallback is the specified contract, not a disagreement between two implementations, so there is no differential axis to push a value through.

## Did I verify this?

- The core `write/2` defect: **yes**, executed (two repro runs in first pass; output preserved above), `verified`.
- Caller behaviour (the first-pass hedge): **yes**, resolved by reading every in-tree call site — mix.ex (install flow + `install_project_config`), fetcher.ex, deps.get.ex, deps.update.ex, deps.unlock.ex. No in-tree caller passes a `:file` diverging from the default; this is the basis for the Medium severity.
- Ruled-out entries: **yes** — both rest on language-level compile/eval facts (`with` bare-clause routing; `@spec` return contract) with no differential input axis, so `Not examined: none` is accurate without needing an executed differential.

### Dropped in re-review

None. The single finding is preserved (severity adjusted High→Medium with cited caller evidence); both ruled-out entries are preserved.
