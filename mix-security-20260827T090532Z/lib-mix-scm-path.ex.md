# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/scm/path.ex
<!-- sha256: 3c70e7b6b21935a37ef5c5b91a635fd43b2879f64fdc00433e30e1df59f497fd -->

_Analysed on 2026-08-27T09:59:48Z_

I have enough to conclude. I read the full 88-line file, traced how `accepts_options/2`, `checked_out?/1`, and `checkout/1` are reached (via `Mix.Dep.Loader.get_scm/2` from `deps` declarations, and directly by `Mix.Dep.Umbrella`), and verified the two path operations' actual behavior against the source-built Elixir.

## Summary
`Mix.SCM.Path` is a thin path-dependency SCM adapter whose only security-relevant operations are unconfined `Path.expand/1` path normalization (intended path-dep semantics) and a benign `File.dir?/1` existence check, with no code-execution, atom-creation, command, deserialization, or write/delete sink present in this file.

## Security

No findings.

### Ruled out

- **Unconfined `Path.expand(opts[:path])` in `accepts_options/2` (line 28).** A dependency declaration `path: "../../../etc"` expands to an out-of-project `:dest` (verified: `accepts_options(:evil, path: "../../../../../../etc")` → `:dest == "/etc"`). This is the documented, intended semantics of a *path* dependency (reference a local directory, relative permitted), and — critically — the dangerous consumption of `:dest` (cd into it, read its `mix.exs`, compile/run its code) lives in **other** files (`mix/dep/loader.ex:339/351`, `mix/tasks/deps.compile.ex:359`). No write/read/exec sink consumes the escaped path *in this file*. Because `fetchable?/0` returns `false` (verified), Mix never downloads anything for a path SCM, so an attacker cannot place code at the escaped path; doing so requires pre-existing local write access, a strictly larger compromise. Not a finding for this file.
- **`checked_out?/1` → `File.dir?(opts[:dest])` (line 59).** The only `File.*` call in the file is a pure directory-existence check (verified: `true` for `/etc`, `false` for a nonexistent dir). At worst a directory-existence oracle over an operator-controlled path — no read of contents, no write/delete/link, no traversal-into. Not a meaningful sink.
- **Umbrella branch `path = "../#{app}"` (line 45) and `Path.expand(path)` (line 48).** `app` is the dependency's application-name atom taken from a `{app, in_umbrella: true}` declaration; interpolating it into a sibling path is the intended umbrella layout. `app` is operator/dependency-declared and the resulting `:dest` again only flows to out-of-file consumers. The two `Mix.shell().error` warnings use `inspect(app)` (no injection). Not a finding.
- **`checkout/1` (lines 78–80).** `Path.relative_to_cwd(opts[:dest])` is pure string manipulation interpolated into a `Mix.raise` message on the terminal "cannot checkout path dependency" error path; no format-string, log-injection, or command sink. Not a finding.
- **Atom creation / code eval / command exec / deserialization / regex / crypto.** None present anywhere in the file (whole file read); `app` arrives already an atom, and no `String.to_atom`, `Code.*`, `System.cmd`, `:erlang.binary_to_term`, or `Regex` call exists.

## Risks / caveats
- The genuinely sensitive behavior — compiling and executing the code found at `:dest` — is **cross-file**. This file only *computes* `:dest`; whether the unconfined value is dangerous depends on the downstream consumers (`mix/dep/loader.ex`, `mix/tasks/deps.compile.ex`, `mix/dep/converger.ex`) and on Mix's convergence rules for whether a **transitive** dependency's path option is honored at all. Those files are out of scope here and were not audited; a reviewer chasing zip-slip-style dep confinement should look there, not at `path.ex`.
- Per the folder threat model, the operator (author of `mix.exs`/umbrella layout) is explicitly out of scope; the only non-operator vector that reaches `accepts_options/2` is a transitive dependency's own `deps`, and even in that case no dangerous sink is reached within this file, so I did not attempt an end-to-end `mix deps.get` reproduction of transitive path-dep handling.
- Reproductions were run against the source-built Elixir at `/opt/elixir` (I confirmed `bin/elixir`, `mix`, `iex` resolve there); results quoted above are `verified`.

