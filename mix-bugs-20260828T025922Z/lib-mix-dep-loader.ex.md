# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/loader.ex
<!-- sha256: 3718bd22d7fa49705acd84e2c452e45e19da7203366c1a6e577b1b1ecb0ef848 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:15:24Z_

## Summary

`Mix.Dep.Loader` is `@moduledoc false` (Mix-internal) but exposes several `@doc`'d functions (`children/1`, `split_by_env_and_target/2`, `skip?/2`, `with_system_env/2`, `load/3`, `vsn_match/3`). I reviewed every function head, `cond`/`case`/`with` branch, and status-tuple flow, and drove the doc'd functions against the build under review; all behave per their documented contract. I found no genuine, reachable bug.

## Bugfixes

No findings.

## Risks / caveats

- The module compiled here is byte-identical in behaviour to the installed `Mix.Dep.Loader` (loading the file only produced a "redefining module" warning), and every documented function I exercised returned the contracted shape. This is a faithful copy of the upstream loader; I found no injected defect.
- Several internal paths (`load/3`, `mix_dep/3`, `rebar_dep/4`, `validate_app/1`, `validate_manifest/1`, `app_status/3`) depend on `Mix.Project`/`Mix.Rebar`/`Mix.AppLoader`/`Mix.SCM` state and a real on-disk dependency tree, so I verified them by static reasoning plus the pieces I could run in isolation (`vsn_match/3`, `skip?/2`, `split_by_env_and_target/2`) rather than a full end-to-end fetch. No contradiction surfaced, but I did not execute a full `deps.get` flow.

### Ruled out

- **`vsn_match/3` `:error` clause on `Version.parse_requirement`** — I suspected a possible `CaseClauseError` if `parse_requirement` returned `{:error, _}` rather than `:error`. Verified against the build under review: `Version.parse_requirement("not a req")` returns `:error` (bare atom), so the `:error ->` clause matches and `Mix.raise` fires correctly. Ran `vsn_match("bad req", "1.2.3", :app)` → raises `"Invalid requirement bad req for app app"`. Not a bug.
- **`vsn_match/3` on invalid `actual` version** — returns `{:error, :nosemver}` before validating `req`; verified with `vsn_match(">= 1.0.0", "not-a-version", :app)` → `{:error, :nosemver}`. This is the documented `:nosemver` status, propagated by `app_status/3` as `{:nosemver, actual}`. Intended.
- **`get_scm/2` empty-keyword truthiness** — `(new = scm.accepts_options(app, opts)) && {scm, new}`: an SCM legitimately returning `[]` (empty opts) is *truthy* in Elixir, so `[] && {scm, []}` yields `{scm, []}` and the SCM is correctly accepted. No accidental fall-through. Not a bug.
- **`attach_only_and_targets/2` merge direction** — `Keyword.merge(merge_opts, opts)` lets a child's own `:only`/`:targets` win over the inherited parent values. This is the longstanding upstream design (conflicting `:only`/`:targets` are caught separately as `:divergedonly`/`:divergedtargets`, which `skip?/2` special-cases at lines 36–37), not a defect introduced here.
- **`validate_manifest/1` clause ordering** — the `_ when not fetchable?` clause precedes the lock/deps-change clauses, so non-fetchable (path) deps skip lock/deps invalidation. The inline comment ("local dep compilation should take care of it") documents this as intended; vsnlock/scmlock are still checked first. Not a bug.
- **`skip?/3` non-atom raise** — the `for entry <- wrapped, not is_atom(entry), do: Mix.raise(...)` raises on malformed `:only`/`:targets`; this is the documented contract for invalid input, not a crash on valid input.
- **`with_scm_and_app/5` `if not Keyword.keyword?(opts)` with no `else`** — when `opts` is a keyword the `if` yields `nil` and execution continues; when it is not, `invalid_dep_format/1` raises and halts. Correct guard behaviour; malformed lists (e.g. `[:foo]`) are rejected as designed.
- **`compile_env_status/2` `with` fall-through** — `nil`/`[]`/valid `compile_env` all route to the `else _ -> {:ok, vsn, properties}` branch; only a present-and-invalid compile env yields `:envoutdated`. The 3-tuple correctly matches `{:ok, vsn, app}` in `validate_app/1`. Behaviour matches the status contract.

