# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.leex.ex
<!-- sha256: b33cdcb8077d1d31847ffc2673551e1fa5d6d0849bebd8db475260a85cb16d86 -->

_Analysed on 2026-08-27T10:37:48Z_

Verified: proplists is first-wins, so `leex_options` can shadow the forced `report/return/scannerfile` opts — but that is the project configuring its own build (same party that controls `mix.exs`), and `do_compile/4` in erlang.ex already has clauses for the degraded `{:ok,_}`/`:error` return shapes, so no cross-boundary or crash impact. I have enough to conclude.

## Summary
`Mix.Tasks.Compile.Leex` is a thin, standard `Mix.Task.Compiler` wrapper that feeds a project's own `.xrl` source files and `:leex_options` into `:leex.file/2`; every input it touches sits at the same trust level as the code being compiled, and it exposes no untrusted-boundary sink of its own.

## Security

No findings.

### Ruled out
- **Output-path traversal via a crafted `.xrl` filename** — The output/`scannerfile` path is `Path.join(src_dir, (Path.basename(input) |> Path.rootname()) <> ".erl")`. `basename |> rootname` collapses every directory component, so the target is always confined to the source dir. Verified: `"src/../../etc/evil.xrl" -> "src/evil.erl"`, `"src/weird/..%2f..xrl" -> "src/..%2f..erl"` (a literal filename, not a parent escape). This pathing logic lives in `Mix.Compilers.Erlang.extract_entries/5`, not in this file, and is not attacker-steerable across a trust boundary regardless.
- **`:leex_options` reaching a dangerous leex option (`scannerfile`, `includefile`, `report`/`return` override)** — `leex_options` comes from `Mix.Project.config()`, i.e. the in-scope project's `mix.exs`. That is the operator/dependency who already runs arbitrary code during their own build; `@recursive true` compiles each dependency under *its own* config into *its own* directories, so a dependency cannot inject options into the parent project's compile. Option precedence is first-wins (verified via `proplists`), so a project could shadow the forced `report/return`, but that only reshapes its own compiler return value — and `do_compile/4` already handles the degraded `{:ok,_}`/`:error` shapes — so there is no crash or cross-boundary write.
- **Compiling a dependency's `.xrl` into generated Erlang** — By design: compiling a dependency already evaluates its `mix.exs` and builds its source, so leex turning that dependency's own `.xrl` into a `.erl` is the same trust level, not a new privilege. The threat model treats build-runs-code as intended.
- **Atom creation (`String.to_unsafe_atom` in `module_from_path`)** — Only reachable inside the `if dest_ext == :beam` branch of `Mix.Compilers.Erlang`; this task uses `dest_ext == :erl`, so it is never invoked — and it is in the other file besides.
- **Manifest `File.read!() |> :erlang.binary_to_term()` without `:safe` (CWE-502)** — Lives in `Mix.Compilers.Erlang.read_manifest/1`, not this file; its input is a file under `_build` (`Mix.Project.manifest_path()`), which requires prior local write access. Not reachable from an untrusted boundary through this file's public API.
- **`project[:app]` interpolated into `Mix.shell().error/1`** in `preload/1` — The value is an atom from the in-scope project's own config, rendered to the developer's stderr; no downstream interpreter and no boundary crossing.

## Risks / caveats
- I settled the path-confinement and leex option-precedence claims by running them in isolation (cited above) rather than by executing a full `mix deps.compile` against a purpose-built hostile dependency; the reasoning that a recursive compile keeps each dependency scoped to its own config/dirs is by inspection of `Mix.Project.config()` + `@recursive true`.
- Two potentially interesting sinks — the unsafe `binary_to_term` manifest read and the `String.to_unsafe_atom` module-name creation — physically reside in `Mix.Compilers.Erlang` (`lib/mix/compilers/erlang.ex`), outside the file under review. They are noted here only to show they were traced; they should be judged in that file (the atom sink is unreachable on the leex `:erl` path, and the manifest read is gated behind local `_build` write access).

