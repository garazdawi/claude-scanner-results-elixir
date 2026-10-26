# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/app.config.ex
<!-- sha256: 010b12f264f2b0a98d0d5e4c464b775431cba48f1cfc19cc542cf04df9796759 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/app.config.ex
<!-- sha256: 010b12f264f2b0a98d0d5e4c464b775431cba48f1cfc19cc542cf04df9796759 -->

_Re-analysed on 2026-08-27_

`Mix.Tasks.App.Config` exposes one documented public entry point, `run/1` (invoked as `mix app.config`). Every sink it reaches operates on operator/developer-controlled inputs (CLI args, `mix.exs` project config, `config/*.exs` files, and the set of already-loaded applications). No dependency-, archive-, network-, or untrusted-file-controlled data reaches any sink here.

The prior report carried two hedges ("did not exhaustively audit `Config.Reader.read!` or `Mix.Tasks.Loadconfig.load_runtime` internals"). I resolved both by reading the callgraph neighbour `lib/mix/lib/mix/tasks/loadconfig.ex` (the module that owns `load_runtime`). See "Did I verify this" for what the read established and what remains genuinely out of file scope.

## Summary

`Mix.Tasks.App.Config.run/1` loads the top-level project's `config/runtime.exs` and warns about non-existent configured apps; every input it touches is operator/developer-controlled, so no untrusted-data path reaches a sink.

## Security

No findings

### Ruled out

- **`Mix.Tasks.Loadconfig.load_runtime(runtime)` (line 41) — code execution.** *Now verified by reading the neighbour.* `load_runtime/1` (`loadconfig.ex:67-71`) calls `Config.Reader.read!(file, env: ..., target: ..., imports: :disabled)` on exactly the single `file` argument it is handed — nothing else. It does not glob, does not follow a directory, and passes `imports: :disabled`, so it cannot be steered to pull in a second (potentially untrusted) config file via `import_config`. The `runtime` argument at `app.config.ex:38` is `config[:config_path] |> Path.dirname() |> Path.join("runtime.exs")`, and `config[:config_path]` originates from the project's own `mix.exs` (operator-controlled, RCE-by-design per the folder threat model). Runtime config is a top-level-only concern (`load_runtime` is invoked only from this task, guarded by `File.exists?/1`), so a dependency gains no new capability beyond the code execution it already has at build time. Confidence: High.
- **Path construction (line 38) — path traversal.** `Path.dirname`/`Path.join` on `config[:config_path]`. The component is operator-controlled from `mix.exs`; there is no untrusted path segment, so `..`/absolute-path confinement is not a boundary concern. The constructed path is consumed only by `File.exists?/1` and then `load_runtime` (which reads that one file, imports disabled — verified above). Confidence: High.
- **`Mix.Task.run("compile", args)` (line 34) and `OptionParser.parse(args, ...)` (line 35).** `args` is the developer's own command line — trusted by definition. Confidence: High.
- **`:code.ensure_modules_loaded(Application.spec(app, :modules))` (line 46) — module loading.** Iterates `Application.loaded_applications()` and loads each app's declared modules. Loading a dependency's modules/code at build time is by-design behaviour already available to any dependency; no new boundary is crossed. A benign self-inflicted race — an app unloaded between `loaded_applications()` and `spec/2` yielding `nil`, then passed to `:code.ensure_modules_loaded(nil)` — could crash the task, but it is not attacker-influenced. Confidence: High.
- **`Mix.shell().error("... #{inspect(app)} ...")` (lines 71–82) — log/terminal interpolation.** `app` is an atom from `Mix.ProjectStack.config_apps()` (operator-controlled config keys). It is emitted through `inspect/1`, which escapes control bytes to literal `\e` sequences rather than raw ESC — verified by running on the built toolchain: `#{inspect(app)}` yields `:"evil\e[31m…"` (escaped) while plain `#{app}` emits the raw escape. No terminal-injection sink, and the source is operator-controlled regardless. Confidence: High.
- **`check_configured/1` guards (lines 62–70) — dead/unreachable defensive code.** The `is_nil(Application.spec(app, :vsn))` and `:code.lib_dir(app) == {:error, :bad_name}` filters are reachable warning logic (fire when a configured app is absent), not a security gate; their only effect is printing an advisory message. Confidence: High.
- **`if !config[:build_path]` (line 57).** A correctness guard against false positives/negatives in umbrella builds; not security-relevant and not a bypassable check. Confidence: High.

## Risks / caveats

- The whole file rests on the (correct, per this folder's threat model) assumption that `mix.exs`, `config/*.exs`, and `config/runtime.exs` are operator-controlled and execute arbitrary code by design. If a reviewer disagreed with that boundary, the `load_runtime` code-execution sink would be reachable — but that would be a codebase-wide re-scoping, not a defect in this file.

## Did I verify this

- **Hedge 1 — `Mix.Tasks.Loadconfig.load_runtime` internals (resolved).** Read `lib/mix/lib/mix/tasks/loadconfig.ex:67-71`. Confirmed `load_runtime/1` reads only its single `file` argument via `Config.Reader.read!` with `imports: :disabled`, then persists the result. This establishes that the only file evaluated is the operator's `config/runtime.exs` derived from `config[:config_path]`, and that no `import_config` redirection to a second file is possible. The trust-boundary placement in the prior report is confirmed, not assumed.
- **Hedge 2 — `Config.Reader.read!` internals (bounded, not a defect).** `Config.Reader` lives in the `elixir` (not `mix`) application and is genuinely outside this folder's file scope. The read of `loadconfig.ex` confirmed the *input* to `read!` is the operator-controlled `file` with imports disabled; whatever `read!` does with that operator file is code execution the operator already authorized. There is no untrusted-data path into `read!` from this file, so no finding is warranted and nothing further needs to be traced to place the boundary. I did not open the `Config.Reader` source; the conclusion does not depend on its internals.
- **Runtime check performed.** The `inspect`-escaping behaviour (lines 71–82) was executed on the built toolchain at `/opt/elixir` (Elixir 1.21.0-dev / OTP 29), matching the source tree under review. All other conclusions are reachability/trust-boundary reasoning verified by reading the two files, not exploitable behaviours requiring reproduction.

### Dropped in re-review

None. No prior finding existed (report was "No findings"); every ruled-out entry is preserved, and the two hedges were resolved to verified ruled-out reasoning rather than upgraded to findings, because reading the neighbour module confirmed the code-execution and path sinks receive only operator-controlled input (imports disabled, single-file read).
