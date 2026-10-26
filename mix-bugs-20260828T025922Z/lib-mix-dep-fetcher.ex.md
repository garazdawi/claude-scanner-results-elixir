# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/fetcher.ex
<!-- sha256: be79646e6ebd1aee4301754ca4ebfe70bdf99350c3f6dd40d80936763553e3d8 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:15:39Z_

## Summary

`elixir/lib/mix/lib/mix/dep/fetcher.ex` is the internal Mix module that drives dependency fetching/updating and lock-file finalization; after close review its logic (fetch gating, out-of-date classification, lock merge order, parent-dep marking) matches the intended upstream behaviour, and I found no reachable correctness defects through its documented entry points (`all/3`, `by_name/4`).

## Bugfixes

No findings

## Risks / caveats

- **Reachability of the public API.** The module is `@moduledoc false`; only `all/3` and `by_name/4` carry `@doc`. I treated those as the entry points. Both delegate the heavy lifting to `Mix.Dep.Converger.converge/4` and `Mix.Dep.*` (out of file), so several behaviours depend on contracts I did not re-verify in those modules (e.g. that `converge` returns a `{all_deps, apps, new_lock}` 3-tuple, and that cyclic dependency graphs are rejected before reaching the fetcher).
- **`String.to_unsafe_atom/1` (line 164).** I confirmed by running the built toolchain (`elixir -e 'String.to_unsafe_atom("zzz_probe_atom")'` → `:zzz_probe_atom`; `String.__info__(:functions)` lists `to_unsafe_atom: 1`) that this is a real, working function in this build, so it is not an undefined-function crash. It creates atoms without the atom-table safety check, but `by_name/4`'s names originate from operator/CLI input (`mix deps.get <app>…`), not untrusted network data, so atom-table exhaustion is not reachable from untrusted input here.
- I did not stand up a full Mix project to exercise `all/3`/`by_name/4` end-to-end (requires a real dep graph, SCMs, and network); the review is source-level plus targeted runtime probes of the ambiguous API.

### Ruled out

- **`Map.merge(old_lock, new_lock)` (line 119) merge direction.** `Map.merge/2` lets the second argument win on key conflict, so `new_lock` correctly overrides `old_lock`, matching the comment "merge the new lock on top of the old." Not a bug.
- **`do_finalize` returns `{apps, all_deps}` (line 125).** `by_name/4` needs the *full* dep list (not the `apps`-filtered `deps`) so its follow-up `Mix.Dep.filter_by_name(names, deps, opts)` can locate (and fail on) named deps that were not among the fetched `apps`. Returning `all_deps` is the correct shape for that check.
- **`mark_as_fetched` ordered before `Mix.Dep.Lock.write` (lines 115/120).** The two operate on disjoint files (`compile.elixir_scm` artifacts vs. the lock file); their relative order carries no correctness dependency, and the ordering matches the stated intent ("Mark parents as fetched before we write the lock file").
- **Potential infinite recursion in `do_with_depending/2` (lines 151–160) on a cyclic dep graph.** A→B→A would loop, but this recursion is guarded by `Enum.all?(all_deps, &available?/1)` and, more importantly, dependency cycles are rejected by `Mix.Dep.Converger` before the fetcher's finalize step runs, so a well-behaved caller cannot deliver a cyclic `all_deps` here. Behaviour is identical to upstream. Not reachable via the documented API.
- **`out_of_date?/1` totality (lines 92–96).** The final `%Mix.Dep{}` clause makes it total over any `Mix.Dep` struct, so no `FunctionClauseError` is possible for the statuses the converger produces.
- **`put_in(dep.opts[:lock], new)` (line 80).** `dep.opts` is a keyword list, which implements `Access`, so the nested `put_in` update succeeds. Not a bug.

