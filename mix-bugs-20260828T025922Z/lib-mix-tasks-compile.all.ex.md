# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.all.ex
<!-- sha256: b6b3d84ce0d8cbfd49f147505270f7e3b85624f7d6e9c67f95a1076df4b37d48 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:39:16Z_

## Summary

`compile.all.ex` is Mix's internal recursive `compile.all` compiler task; it orchestrates app-cache computation, code-path pruning, running the configured compilers, and compile-env validation. After tracing every branch and verifying the one plausible crash (charlist interpolation), I found no genuine reachable defects.

## Bugfixes

No findings

## Risks / caveats

- I verified the only candidate crash — the charlist path built at line 89, `"#{compile_path}/#{app}.app"`, where `compile_path` is a charlist (`to_charlist/1`, line 62). `String.Chars.List` is implemented, so interpolation yields the intended string rather than raising. Confirmed by running `elixir -e 'cl = to_charlist("/tmp/foo"); IO.puts("#{cl}/#{:bar}.app")'` → `/tmp/foo/bar.app`.
- The review rests on the documented contracts of the collaborators (`Mix.AppLoader.stale_cache/load_apps/load_app/write_cache`, `Mix.Dep.cached`, `Mix.Task.Compiler.run`, `Mix.Utils.project_apps`, `Config.Provider.validate_compile_env`, `Mix.Project.with_build_lock`), which live in other files and were not re-audited here; a contract break in one of them could surface through this task but would be a bug in that file, not this one.

### Ruled out

- **`with` block (lines 89–94) silent wrong return** — the `with` has no `else`, so an unmatched clause returns the failing clause's value, but that value is discarded (the following statement `{status, diagnostics}` at line 97 is the function result). Each early-exit path is correct: flag present (`false <- "--no-validate-compile-env" in args`) skips validation; nil/empty `compile_env` (`[_|_] = compile_env <- ...`) skips; `:ok` from `validate_compile_env` skips; only a genuine `{:error, message}` reaches `Mix.raise`. Traced each branch by hand.
- **`app_cache`/`loaded_modules` threading** — `app_cache` is `nil` when `--from-mix-deps-compile` is present (bare `if` with no `else`), and `loaded_modules` is only accumulated when `app_cache` is truthy (line 45), so `write_cache` (line 82) is guarded by the same `app_cache` condition. Consistent.
- **`exit({:shutdown, 1})` leaking the build lock (lines 22, 79)** — the `exit` occurs inside `Mix.Project.with_build_lock`'s callback; `exit` is an exception-class signal, so a `try/after` release in `with_build_lock` runs. This is also the intended failure path for a compile error without `--return-errors`. Not re-verified against `with_build_lock`'s source, but the pattern is standard and the behavior is by design.
- **`compile_apps/2` optional-deps subtraction (lines 120–124)** — `all -- optional` only when `--no-optional-deps` present; `all`/`optional` come from `Mix.Utils.project_apps`'s documented `{all, optional}` shape. No shape mismatch.
- **`Map.new(loaded_modules)` (line 83)** — `loaded_modules` is a list of `{app, modules}` two-tuples built at line 45, a valid input for `Map.new/1`.

