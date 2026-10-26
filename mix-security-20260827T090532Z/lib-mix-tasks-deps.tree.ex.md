# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.tree.ex
<!-- sha256: 2d66bff32faf62a2e2395b8d402060c2c60b7036dc0e3487eccd1be5d7a2d643 -->

_Analysed on 2026-08-27T10:52:18Z_

The task runs from the source under review and behaves exactly as traced: the only external inputs are CLI arguments, and `this_dep_does_not_exist_xyz` was converted to an atom (via `String.to_unsafe_atom`) then discarded on lookup failure. My analysis is complete.

## Summary
`Mix.Tasks.Deps.Tree` reads only developer-supplied command-line arguments and already-converged dependency metadata, formats them into display strings, and delegates all file/terminal output to `Mix.Utils` (which escapes DOT output); no attacker-controlled data reaches a dangerous sink inside this file.

## Security

No findings.

### Ruled out

- **`String.to_unsafe_atom/1` on `--only`/`--target` (line 88), positional app (line 99), and `--exclude` (line 130)** — `String.to_unsafe_atom/1` is the un-deprecated successor to `String.to_atom/1` and is exactly `:erlang.binary_to_atom(string, :utf8)` (verified at `/opt/elixir/lib/elixir/lib/string.ex:3040`), so these are genuine dynamic atom-creation sinks (CWE-770). But every value is a command-line argument the developer types when invoking `mix deps.tree` (verified: the `run/1` entry point is fed only `args`; `this_dep_does_not_exist_xyz` was converted then raised). CLI arguments are the trusted operator's own input — not a dependency, archive, git ref, network peer, or a file the project reads — so they do not cross the trust boundary this codebase cares about. Structurally, exhaustion is also impossible here: a handful of atoms are created per short-lived, per-invocation BEAM, with no long-lived accumulation, and the resulting atom is used only for an `Enum.find` lookup / display, never reaching `apply/3` or a process name.
- **`scm.format(opts)` dispatch (line 192)** — `scm` is a `Mix.SCM` module resolved by Mix from its registered SCM set (Git/Path/Hex), not an arbitrary module a dependency can set; the callees are trusted Mix code. Not attacker-controlled module dispatch, and cross-file regardless.
- **`requirement/1` → `inspect(regex)` / binary pass-through (lines 195–196), and formatted node strings reaching `Mix.shell().info`/`print_tree`/`write_dot_graph!`** — a malicious dependency's `requirement`/app metadata is dependency-controlled, but the only sinks are in `Mix.Utils`/`Mix.Shell` (out of this file's scope); the DOT path escapes every name and label via `quoted/1` → `escape_dot_string/2`. Moreover, dependency convergence evaluates each dep's `mix.exs` as code, so any malicious dependency already has RCE on the build machine before `deps.tree` prints anything — terminal-escape injection would be strictly weaker than an exposure that already occurred.
- **`--output` path handling / `.bak` rename (lines 106–113; `Mix.Utils.write_according_to_opts!`)** — the output path and its documented in-place `.bak` clobber are developer-chosen; `Path.rootname`/`basename`/`relative_to_cwd` only build a display hint (`inspect`-quoted, not executed). Self-inflicted and cross-file.

## Risks / caveats
- I reviewed only this file. The real security-relevant behaviour it depends on lives elsewhere and was not audited here: DOT/terminal escaping and the `--output`/`.bak` file write in `Mix.Utils`, and — most importantly — the arbitrary-code execution and atom creation performed by `Mix.Dep.Converger.converge/1` when it loads each dependency's `mix.exs`. Any terminal-escape-injection or atom-exhaustion concern arising from hostile dependency metadata belongs to those files (`Mix.Dep.Loader`, `Mix.Shell.IO`), not to `deps.tree.ex`.
- Minor non-security note (not a finding): the moduledoc examples reference `--include` (lines 65–67), but `@switches` (lines 71–78) never defines it, so `--include foo` is silently parsed as a boolean flag with `foo` demoted to the positional root app. This is a documentation/UX bug with no security impact.
- Reproductions ran under the sandbox's `Elixir 1.21.0-dev` / OTP 29 built from the source under review; the file is byte-identical to `/opt/elixir`'s copy (`diff` exit 0), so no injected modification is in play.

