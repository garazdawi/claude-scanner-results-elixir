# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.all.ex
<!-- sha256: b6b3d84ce0d8cbfd49f147505270f7e3b85624f7d6e9c67f95a1076df4b37d48 -->

_Analysed on 2026-08-27T10:28:19Z_

I have completed my analysis. The build environment works (Elixir 1.21.0-dev / OTP 29, built from the source under review), and the one non-trivial control-flow gate behaves as intended.

## Summary

`Mix.Tasks.Compile.All` is compile-orchestration glue whose every dangerous operation is delegated to other modules and whose inputs (`args`, `config`, `project`, `deps`) are all operator-controlled by design, so no in-file security finding survives tracing.

## Security

No findings.

### Ruled out

- **`"#{compile_path}/#{app}.app"` path interpolation (line 89).** Both interpolated components are operator-controlled: `compile_path` is `Mix.Project.compile_path()` (the local build dir) and `app` is `config[:app]` from the current project's `mix.exs`. No dependency/archive/network-derived component reaches this path, and the actual file read/parse sink lives in `Mix.AppLoader.load_app`, not here. Not a traversal from an untrusted boundary.
- **Code-path prepend of dependency paths (lines 43–57).** Dependency ebin paths flow from `Mix.Dep.cached()` / `Mix.AppLoader.load_apps` into `Code.prepend_paths`. Making a dependency's compiled modules loadable is the documented purpose of compilation (dependency code executing is the by-design "build scripts run" boundary from the folder threat model), and `Code.prepend_path` adds a search directory — it is not a file-write/traversal sink. No additional privilege is crossed.
- **Code-path pruning / archive-SCM hardening (lines 53–55).** `prune_code_paths` defaults to `true` and the `--no-prune-code-paths` override comes from the top-level `mix` invocation; a dependency compiled recursively only supplies its *own* config to its *own* pass and cannot suppress pruning for the parent. The pruning `current_paths -- loaded_paths` correctly targets paths not belonging to the current project's deps (e.g. installed-archive SCMs). Working as intended.
- **compile-env validation `with` chain (lines 86–95).** This is the only security-relevant gate. Reproduced its exact guard semantics (`elixir -e` above): validation failure → `Mix.raise` fires; validation `:ok` → skip; `--no-validate-compile-env` → skip; empty/absent env → skip; missing `.app` → skip. The gate is reachable through a normal `mix compile`, fires correctly, and is not dead code; its only bypass is the explicit developer flag `--no-validate-compile-env`, which is a documented opt-out, not an attacker-reachable weakening.
- **`compile_apps/2` (the documented public function, lines 103–125).** Reads `project.application()` and `config`, both operator-controlled; `args` is used only in membership tests (`"--no-optional-deps" in args`), never interpolated into a path/command/atom. `Mix.Utils.project_apps` is the real sink and lives in another file. Reaching a hostile result would require a caller to pass a hostile `config`/`args`, which is excluded misuse of the contract.
- **Atom / eval / command / deserialization sinks.** None present in this file: no `String.to_atom`/`Module.concat`, no `Code.eval*`/`Code.compile*`, no `System.cmd`/`System.shell`/`:os.cmd`, no `binary_to_term`, no `:erl_tar`/`:zip`. Compiler execution (`Mix.Task.Compiler.run/2`) runs operator-configured compilers.

## Risks / caveats

- I reviewed only this file per scope. The genuine sinks it drives — `.app` term parsing (`Mix.AppLoader.load_app`), stale-cache/cache serialization (`Mix.AppLoader.stale_cache`/`write_cache`), app enumeration (`Mix.Utils.project_apps`), and the compiler run itself (`Mix.Task.Compiler.run`) — live in other modules and were not audited here; if a dependency-controlled `.app`/manifest is parsed unsafely, that finding belongs to those files.
- I did not execute a full end-to-end `mix compile` inside a crafted project (heavy, and no positive finding required a live repro); the reproduction I ran isolates and confirms the one subtle control-flow gate. The conclusion rests on data-flow tracing showing every hostile-input class terminates in another file or in an operator-trusted value.

