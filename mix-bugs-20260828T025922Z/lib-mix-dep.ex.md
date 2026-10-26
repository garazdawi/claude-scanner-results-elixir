# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep.ex
<!-- sha256: faca7f2a34792c70c4eb1d0e7a0a87effdc1d72ce7e2dbe27601ee09128f59ed -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:13:02Z_

## Summary

`mix/dep.ex` is the `Mix.Dep` struct plus its query/format helpers (loading & caching deps, status formatting, lock checking, predicate accessors). Reviewing every documented public function, I found no reachable correctness defect: `format_status/1` is total over every status tag the codebase actually produces, the predicates are exhaustive, and `in_dependency/3` restores env state via `try/after`.

## Bugfixes

No findings.

## Risks / caveats

- `format_status/1` totality was verified two ways: (1) I rendered all 14 status shapes a loaded dep can carry and every one produced a string with no `FunctionClauseError` (`elixir t.exs`, output quoted below), and (2) I grepped the whole `mix/dep*` and `tasks/deps*` tree for status tags actually constructed — the set is exactly `{:ok, :noappfile, :invalidapp, :invalidvsn, :nosemver, :nomatchvsn, :lockmismatch, :diverged, :divergedreq, :divergedonly, :divergedtargets, :overridden, :unavailable, :vsnlock, :scmlock, :depschanged}` plus atoms `:compile/:lockoutdated/:nolock/:envoutdated`, all of which have a matching `format_status` clause. Rendered output:
  ```
  {:ok, "1.0.0"} => ok
  {:noappfile, {"/a/ebin/x.app", nil}} => could not find an app file ...
  {:unavailable, "/p"} => the dependency is not available
  {:vsnlock, 1} => the dependency was built with an out-of-date Erlang/Elixir version ...
  ... (all 14 rendered without error)
  ```
- I did **not** exhaustively trace `Mix.Dep.Converger.converge/1` (out of file); the ruling that `format_status` is total rests on the grep above finding no additional tags in the tree, not on reading the converger line-by-line.
- The private umbrella path `load_and_cache/5` (cache-read + child recomputation) is intricate but only reachable through `load_and_cache/0` in a multi-project stack, which I could not stand up in isolation here; I reasoned through its accumulator/`tl` logic (no infinite recursion — `seen` is seeded with the roots before every `get_children` call) but did not execute it end-to-end.

### Ruled out

- **`to_app_names/1` uses `String.to_unsafe_atom` on `filter_by_name/3` names (atom-table growth).** Reachable via `mix deps.get foo` etc., but the names are operator/CLI-controlled (not network input), the standing threat model explicitly lists this as an accepted existing pattern, and unknown names are immediately rejected with `Mix.raise("Unknown dependency …")`. Not a novel defect.
- **`format_status/1` / `load_paths/1` raising on `status: nil` or `opts[:build] == nil`.** Only reachable by passing an *unloaded* dep, which violates the documented "expects a loaded dependency" contract — out of scope per the contract-violation rule.
- **`check_lock/1` `case scm.lock_status(opts)`** only handling `:mismatch | :outdated | :ok`. Verified against the `Mix.SCM` behaviour contract (`lock_status/2` documents exactly those three returns); no other value is producible by a conforming SCM.
- **`get_children/3` infinite recursion on a dependency cycle.** `seen` is populated with the current apps (via `populate_seen`) before recursing, and both call sites seed `seen` with the roots first, so a cycle terminates. Traced by reading the recursion; the `filter_by_name`/`load_and_cache` seeding is present at lines 137 and 207.
- **`format_dep/1` double-space when version is empty.** `"#{app} #{version}(...)"` with `version = ""` yields `"foo (…)"` — a single space, which is the intended rendering.
- **`in_dependency/3` leaking the mutated `Mix.env`.** Restored in `after` (line 281), satisfying the resource/state-cleanup obligation even if `fun` raises.

