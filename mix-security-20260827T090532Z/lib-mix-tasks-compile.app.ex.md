# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.app.ex
<!-- sha256: 519911e2b2ef306b3351ace0030a495aabb0c6ab123800c163939ad41833ae12 -->

_Analysed on 2026-08-27T10:33:56Z_

I now have a complete, verified picture. Every potentially-untrusted input to `run/1` is reachable only after the relevant `mix.exs` has already executed as arbitrary code. Here is the report.

## Summary

`Mix.Tasks.Compile.App.run/1` (the `compile.app` build task) contains atom-creation, file-write, and `:file.consult`/`:application.load` sinks, but every one of them is fed only by data (project/dependency `mix.exs` config, compiled `.beam` filenames, the on-disk `<app>.app` file) that becomes attacker-controllable only after that project's or dependency's `mix.exs` has already run as arbitrary code — so no finding crosses a trust boundary that isn't already an RCE boundary by Mix's documented design.

## Security

No findings.

### Ruled out

- **Atom-table exhaustion / crash via `modules_from/1` → `String.to_unsafe_atom` (line 266).** `String.to_unsafe_atom/1` is `:erlang.binary_to_atom(bin, :utf8)` (verified at `string.ex:3040`), which creates a never-collected atom per `.beam` filename and raises `SystemLimitError` on names >255 code points (both verified: a runtime-built stem produced `delta 1` new atom; a 300-char stem raised `SystemLimitError`). However, `compile_path` is the current project's/dependency's own `ebin`, populated by compiling that project — which requires its `mix.exs` (and any extra `.beam`-writing build script) to have already executed arbitrary code on the build machine. The folder threat model explicitly places "dependency build scripts run arbitrary code" out of scope; the sink grants no capability the file-planting attacker lacks. This is stock, decade-old Mix code (installed `compile.app.ex` matches byte-for-byte), not a plant.
- **Atom creation / term DoS via `current_app_properties/1` → `:file.consult(target)` (line 231).** Parsing the `<app>.app` file interns an atom for every atom literal and reads the whole file into terms. `target` lives under `compile_path` (`_build/...`), writable only by a process that has already achieved code execution on this machine. Not `binary_to_term`, so no funs/pids/arbitrary structures — just the Erlang term reader.
- **`:application.load({:application, app, current_properties})` / `.load(... properties)` (lines 204–205, 225) with parsed-from-disk properties.** `load` only registers application metadata (env/mod), it does not start anything; and the properties originate from the RCE-gated `.app` file or the project's own `application/0`. No start-time callback fires here.
- **`File.write!`/`File.touch!` to `Path.join(compile_path, "#{app}.app")` (lines 209–221).** `app` is validated `is_atom` and `compile_path` comes from `config`/`--compile-path` — both the developer's own trusted invocation or an already-RCE dependency config; no non-RCE attacker controls either component, so no path traversal boundary is crossed.
- **`project.application()` invocation (line 276).** Guarded by `function_exported?/3`; it calls the project's *own* code, which is the definition of the trusted-operator boundary, not external-input code execution.
- **`validate_app!` / `validate_version!` / `validate_properties!` / `typed_app?`.** These are developer-facing shape validators over the project's own `mix.exs` config, not safety predicates over attacker input; all clauses (including the non-atom `app` and non-`{module,args}` `:mod` fallbacks) are reachable and fail closed via `Mix.raise`. No dead defensive code.
- **Interpolation into `Mix.raise`/`Mix.shell().info|error`.** Values are `inspect/1`-rendered or the atom `app`; no downstream interpreter (shell/SQL/regex/EEx) is reached.

## Risks / caveats

- I confirmed the atom-creation behaviour empirically (`:erlang.binary_to_atom` creates a new atom, `delta 1`; >255 chars raises `SystemLimitError`) but did not stand up a full malicious-dependency `mix compile` end-to-end; the reachability conclusion rests on the documented Mix build model (dependency `mix.exs` executes before `compile.app` runs recursively for that dep) rather than a live exploit attempt, which is appropriate since the path is RCE-gated regardless.
- Only `lib/mix` is present in the review tree; helper semantics (`Mix.Utils.project_apps`, `Mix.Utils.consultable`, `String.to_unsafe_atom`) were confirmed against the installed source at `/opt/elixir`, which matches the file under review. `consultable` was checked to be pure term serialisation with no eval/network/command/string→atom sink (it even rejects compiled regexes).
- `Mix.Project.*`/`Mix.ProjectStack.*` state accessors are treated as trusted local build infrastructure per the folder threat model; I did not audit those modules here.

