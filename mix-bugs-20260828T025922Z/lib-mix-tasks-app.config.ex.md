# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/app.config.ex
<!-- sha256: 010b12f264f2b0a98d0d5e4c464b775431cba48f1cfc19cc542cf04df9796759 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:31:02Z_

## Summary

`app.config.ex` is a small Mix task (`Mix.Tasks.App.Config.run/1`) that compiles the project, loads `config/runtime.exs`, optionally preloads modules, and warns about configured-but-missing apps. I found no reachable behavioural defects.

## Bugfixes

No findings

## Risks / caveats

- The task's real work happens through `Mix.Task.run("compile", args)`, `Mix.Tasks.Loadconfig.load_runtime/1`, `Mix.Project.config/0`, and `Mix.ProjectStack.config_apps/0` — all defined in other files and out of scope here. The review assumes those honor their documented contracts (e.g. `config[:config_path]` is always populated with the `"config/config.exs"` default, so `Path.dirname/1` never receives `nil`).
- I could not stand up a full Mix project in the sandbox to drive `run/1` end-to-end, so findings rest on static reasoning plus knowledge of the OTP/Mix APIs rather than an executed reproduction. No claim of absence below depends on an untested behavioural difference.

### Ruled out

- **`Application.spec(app, :modules)` returning `nil` in the `--preload-modules` branch** — the comprehension iterates `Application.loaded_applications()`, so every `app` is already loaded; `Application.spec/2` only returns `nil` for *unloaded* apps. For loaded apps the `:modules` key is always a (possibly empty) list, which `:code.ensure_modules_loaded/1` accepts. No crash path.
- **`:code.ensure_modules_loaded/1` arity/existence** — exists since OTP 21 and takes a module list; the argument supplied is exactly a module list. Correct.
- **`OptionParser.parse` with only `preload_modules` in `@switches`** — the other documented flags (`--force`, `--no-compile`, `--no-deps-check`, etc.) are intentionally forwarded verbatim via `args` to the `compile` task and are not meant to be parsed here. Unknown switches are placed in the (discarded) invalid list, not raised. Intended behaviour, not a bug.
- **`runtime` path build `config[:config_path] |> Path.dirname() |> Path.join("runtime.exs")`** — with the default `"config/config.exs"` this yields `"config/runtime.exs"`; guarded by `File.exists?/1` before loading. No traversal or nil-arg issue given a well-formed config.
- **`{app, _, _}` match on `Application.loaded_applications()`** — that function's documented return element shape is exactly `{atom, charlist, charlist}`; the match is total over its output.
- **`check_configured/0` filter (`is_nil(spec(app,:vsn))` + `lib_dir == {:error, :bad_name}`)** — both comparisons are total over their possible return values (nil / vsn string, and `{:error, :bad_name}` / path charlist); the `for` simply emits an error message and returns `:ok`. No missing clause or wrong-shape match.

