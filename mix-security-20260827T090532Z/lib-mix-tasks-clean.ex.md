# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/clean.ex
<!-- sha256: c6a82c163abcf14c6ff5796acd2eca105fd3cda1f5738900bbcc98f9882f52a8 -->

_Analysed on 2026-08-27T10:25:46Z_

## Summary

`Mix.Tasks.Clean` is a small, self-contained build-artifact deleter whose every sink (`Path.wildcard`, `File.rm_rf`, the `compile.#{compiler}` → `Module.concat` atom path, and `&1.clean()` dispatch) draws exclusively from operator-controlled inputs — command-line args, `mix.exs`/config, and env vars — with no path from a dependency, archive, git ref, or network peer, so there is no attacker-reachable vulnerability.

## Security

No findings

### Ruled out

- **`Path.wildcard("*#{opts[:only]}")` + `File.rm_rf/1` (lines 57–67) — path/glob injection into a destructive delete.** The only variable component, `opts[:only]`, comes from `OptionParser.parse(args)` where `args` is the argument to `run/1`, i.e. the developer's own command line (`--only`). The confinement root, `Mix.Project.build_path() |> Path.dirname()`, resolves from project config / `MIX_BUILD_PATH` (`project.ex:769`) — all operator-controlled. No dependency/archive/git/network path sets `--only` or `build_path`. Per this codebase's severity model ("the developer trusts their own code is true and uninteresting"), a developer choosing a destructive `--only` value on their own shell is not a security boundary crossing.
- **`Mix.Task.get("compile.#{compiler}")` → `command_to_module` → `Module.concat` (line 47) — atom-table injection (CWE-770).** `command_to_module` (utils.ex:267) uses unchecked `Module.concat`, which I confirmed creates a new atom per novel name (`atom_count delta: 1` from the run above). However `compiler` originates from `Mix.Task.Compiler.compilers()` = `config[:compilers] || Mix.compilers()` (task.compiler.ex:218) — the current project's own `mix.exs` config. Setting `:compilers` already implies operator RCE at build time, and a dependency cannot inject into the parent's `:compilers`, so the atom-creating input never crosses an attacker boundary.
- **`Enum.each(tasks, & &1.clean())` and `Mix.Compilers.Protocol.clean()` (lines 51–52) — dynamic behaviour dispatch / code execution.** The dispatched modules are the compilers listed in the operator's config; those same modules' `run/1` already execute arbitrary code during `mix compile`, so invoking their `clean/0` grants no new privilege and is gated by `function_exported?/3`.
- **`Path.join("lib/#{Mix.Project.config()[:app]}")` (line 65).** The `:app` name is from `mix.exs`, operator-controlled.
- **`loadpaths!/0` (lines 72–83).** Delegates to the `loadpaths` task with hardening flags; any sink lives in that task's file, out of scope for this review. It adds code paths (no dependency-code execution) before the deletes.
- **Concurrency around the deletes.** `do_run` runs inside `Mix.Project.with_build_lock/1` (line 37); no attacker-influenced race.

## Risks / caveats

- I did not execute a full `mix clean` end-to-end against a crafted umbrella/dependency tree; the trust-boundary conclusions for `config[:compilers]`, `config[:app]`, and `build_path` are established by reading their definitions (`task.compiler.ex:217`, `project.ex:769`) and confirming `@recursive` descends only into operator-owned umbrella children, not into fetched dependencies. I confirmed by running that the `--only` interpolation and the `Module.concat` atom creation behave as described.
- Sinks reached *through* this file but defined elsewhere (`Mix.Task.run("loadpaths", …)`, `Mix.Compilers.Protocol.clean/0`, `Mix.Task.get/command_to_module`) were traced only far enough to establish their inputs are operator-controlled; a vulnerability internal to those modules, if any, belongs to their own files.

