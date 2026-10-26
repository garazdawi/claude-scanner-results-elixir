# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.compile.ex
<!-- sha256: db16c430499057cd2f89a8575c76e976ca7366d6afbb03129c02a333e1dc737e -->

_Analysed on 2026-08-27T10:47:12Z_

## Summary
`Mix.Tasks.Deps.Compile` (`mix deps.compile`) orchestrates building already‑fetched dependencies, and every dangerous operation it performs is either by‑design execution of dependency build logic or manipulation of trusted build paths — so no in‑scope, file‑specific vulnerability was found.

## Security

No findings

### Ruled out

- **`File.rm_rf!(Path.join([build_path, "lib", Atom.to_string(dep.app)]))` (line 129) — path traversal via app name.** `dep.app` is an atom that reaches this file inside a `Mix.Dep` struct built by `Mix.Dep.Loader`; `with_scm_and_app/5` only guards `is_atom(app)` and never rejects textual `..`/`/`, so a traversal atom (e.g. `:"../../pwned"`) would compose a traversal path — verified: `Path.join(["_build/dev","lib", Atom.to_string(:"../../../pwned")])` → `_build/dev/lib/../../../pwned`. Rejected because the same atom already drives `opts[:build]`/`opts[:dest]` used by every other file op here, its provenance is upstream‑constrained (top‑level deps names are author‑controlled; the only externally‑sourced names come from a dependency's `rebar.config`, whose sibling `rebar.config.script` is executed via `:file.script` at load time — i.e. that dependency already has arbitrary code execution), and it must first survive `deps.get` placing it under `deps/<traversal>`. The traversal adds no capability beyond the by‑design RCE, so it is a cross‑file, upstream concern rather than an incremental defect in this file.
- **`opts[:compile]` → `System.shell` (do_compile → shell_cmd!, lines 336/346).** `Mix.Shell.cmd/3` runs a binary command via `System.shell/2` (confirmed in `shell.ex:164`). This is the documented `compile: "command"` feature; the command originates from the dep declaration (author for the top‑level, dependency‑as‑code for transitive deps). Running a dependency's build command is exactly the intended, threat‑model‑sanctioned behaviour.
- **`do_make`/`make_command` shell string (lines 306–333, 346).** The command is a fixed literal (`make`/`gmake`/`nmake /F Makefile.win`, or `"#{cmd} clean && #{cmd}"` where `cmd` is itself one of those literals). No dependency‑controlled data is interpolated into the shell string; the only dep influence is `erlang.mk`/`Makefile.win` file existence. Executing the dep's Makefile is by design.
- **`do_rebar3` command (line 283).** Passed as `{exec, args}`, so `Mix.Shell.cmd/3` uses `System.cmd/3` (argv, no shell) — the `lib_path` glob and paths cannot be shell‑injected. Invoking rebar to build a rebar dep is by design.
- **`MIX_OS_DEPS_COMPILE_PARTITION_COUNT` → `String.to_integer/1` (line 103).** The env var is operator/CI‑controlled, not attacker‑reachable; a non‑integer value only crashes the caller's own build (self‑inflicted).
- **`Code.prepend_path`, `File.cp_r`, `File.mkdir_p!`, `Mix.Utils.symlink_or_copy`, `File.write!(mix.rebar.config)` (lines 255–296, 362–366).** All operate on build/dest paths derived from the project config and the app atom (fixed filenames under `opts[:build]`); the rebar config *content* comes from `dep.extra` but is serialized by `Mix.Rebar` and consumed by rebar, which the dependency controls anyway.
- **`deps_compile_feedback(app)` interpolating `app` into messages (lines 420–429, 285, 348).** The result is only printed/`Mix.raise`d, never executed; `app` is an atom.
- **`scm.fetchable?()` / `scm.*` dispatch (lines 108, 184, 203, 394, 400, 413).** `dep.scm` is a Mix/Hex SCM module chosen by the loader's `get_scm`/`accepts_options`, not an attacker‑supplied module name.

## Risks / caveats
- This file contains **no** network fetch, checksum verification, archive/tarball extraction, atom‑from‑external‑bytes, or `binary_to_term` — those live in the SCM/`deps.get`/archive tasks, so the classic transport/zip‑slip/deserialisation sinks are out of this file's scope by construction.
- The central caveat (`File.rm_rf!` / build paths lack `Path.safe_relative` confinement on the app‑name component) is genuine but its exploitability is decided in **other files** (`Mix.Dep.Loader`, `Mix.Rebar`, `deps.get`): I confirmed there is no textual validation of app‑name atoms in the loader path shown, but confirming an end‑to‑end traversal would require driving a full `deps.get` + converge with a crafted rebar dependency, which I did not execute here. Given that any dependency able to supply such a name also executes `rebar.config.script` at load time, I judged it not to add incremental capability and did not build the full repro.
- Behavioural claims about shell vs. argv dispatch and `Path.join` traversal were verified by reading `shell.ex` and running Elixir; the by‑design nature of dependency build execution is taken from the folder threat model rather than re‑derived.

