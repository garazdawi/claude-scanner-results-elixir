# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/loadconfig.ex
<!-- sha256: a20aeefc6f76b93f8a4cd21851afd5b3bd44ebdb254bacdaa986b8ebc65f62b8 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/loadconfig.ex
<!-- sha256: a20aeefc6f76b93f8a4cd21851afd5b3bd44ebdb254bacdaa986b8ebc65f62b8 -->

_Re-reviewed on 2026-08-27_

## Summary

`Mix.Tasks.Loadconfig` loads and persists Elixir config files. Its only powerful sink is config evaluation (`Config.Reader.read_imports!`/`read!`), which is by-design compile-time code execution on operator-controlled files. I re-verified every in-repo caller by reading the neighbour modules: all pass an operator-controlled path (CLI argv, the project's own `:config_path`, the derived `config/runtime.exs`, `--config` argv, or `~/.mix/config.exs`). No attacker-influenced path (dependency config, archive content, network body, git ref) reaches this sink. No security findings.

## Security

No findings.

### Ruled out

- **Config file evaluation is arbitrary code execution (`load_compile`/`load_runtime`, lines 59–71).**
  `Config.Reader.read_imports!`/`read!` evaluate the config file as Elixir. **Reproduced** on Elixir 1.21.0-dev / OTP 29 (the toolchain the source tree targets): a `config.exs` containing `File.write!("/tmp/pwned_by_config", ...)` executed at read time — `side-effect file exists?: true`, `config: [foo: [bar: 1]]`. This is documented, intended compile-time-config behavior. I read every in-repo caller and confirmed each supplies an **operator-controlled** path:
  - `run/1` line 38 — the `mix loadconfig <file>` argv.
  - `load_default` line 46 — the project's own `Mix.Project.config()[:config_path]` from `mix.exs`.
  - `app.config.ex:38–41` — `runtime` is derived as `config[:config_path] |> Path.dirname() |> Path.join("runtime.exs")`, i.e. the operator project's own `config/runtime.exs`; fed to `load_runtime`. **Verified by read.**
  - `archive.install.ex:155` and `escript.install.ex:135` — `Mix.Task.run("loadconfig")` with **no file arg**, so it hits `load_default` → project config_path, not any archive/escript content. **Verified by read.**
  - `local/installer.ex:106` — `preserve_config(fn -> module.build(...) end)`; `preserve_config` reads the already-loaded project config via `read_compile()` and does not itself read any new file. **Verified by read.**
  - `run.ex:184` — the deprecated `--config` flag value (operator argv).
  - `cli.ex:264` — the user's `~/.mix/config.exs` (operator home).
  Mix does not load a *dependency's* config files; no dependency/archive/network/git ref reaches this sink. A malicious `mix.exs` already has RCE by design (out of scope per folder threat model).
  Axes examined: caller-supplied path provenance (all 7 in-repo call sites read); eval side-effect (executed). | Not examined: none.

- **`run/1` missing catch-all clause (lines 36–39).**
  `mix loadconfig a b c` (2+ args) matches neither `[]` nor `[file]`, raising `CaseClauseError`. **Reproduced** on Elixir 1.21.0-dev / OTP 29: `CRASH: CaseClauseError`. Args are operator-typed on the command line; benign usage error, not attacker-reachable.
  Axes examined: arg-list arity 0/1/≥2 (executed). | Not examined: none.

- **TOCTOU on `config_path` (line 45) and `~/.mix/config.exs` (`cli.ex:263`).**
  `File.regular?(config_path)` is checked, then `load_compile` re-reads the path. Any racer able to swap the file is already inside the operator's trust domain (their own project dir / home dir); no in-scope attacker controls these paths.
  Axes examined: path provenance (project dir / operator home). | Not examined: cross-user race on a world-writable config dir — self-inflicted misconfiguration only, no in-scope remote/dependency attacker reaches the path, so no executable trigger exists.

- **Global-state writes: `Application.put_all_env(config, persistent: true)` (line 99), `Mix.State.write_cache` (line 62), `Mix.ProjectStack.loaded_config` (line 61).**
  These persist config into VM-global application env, but the values come from the operator's own config file (established above), not an external boundary.
  Axes examined: value provenance (operator config, per verified callers). | Not examined: none.

- **`persist_apps` persists reserved apps before warning (lines 98–134).**
  `:kernel`/`:stdlib` config is written by `put_all_env` (line 99) *before* the "Cannot configure base applications" check (line 102), which is only an advisory `Mix.shell().error`, not a security gate. Config content is operator-controlled, so no privilege boundary is crossed.
  Axes examined: ordering (put_all_env precedes the filter); value provenance (operator config). | Not examined: none.

- **Error-string interpolation (lines 108, 131): `inspect(reserved_apps)` and `Path.relative_to_cwd(file)`.**
  Both flow only into a `Mix.shell().error` message rendered to the operator's terminal; no downstream interpreter (shell, query, EEx, regex) consumes them.
  Axes examined: sink type (terminal `error/1` only, no secondary interpreter). | Not examined: none.

- **No atom-from-bytes, command execution with external argv, archive extraction, `binary_to_term`, crypto, or outbound network** exists in this file. (Confirmed by read of the full 138-line file.)

## Risks / caveats

- The security posture rests on the invariant that **Mix never loads a dependency's config files** — only the top-level project's config, the user's global config, and operator-supplied paths flow into `load_compile`/`load_runtime`. I re-verified this by reading all in-repo call sites (`cli.ex`, `run.ex`, `app.config.ex`, `archive.install.ex`, `escript.install.ex`, `local/installer.ex`, `compilers/elixir.ex`); every one passes an operator-controlled path or no path at all. This is now a verified claim, not an assumption.
- The `@doc false` functions `load_compile/1`, `load_runtime/1`, `read_compile/0`, and `preserve_config/1` are exported and thus technically callable by out-of-tree tooling. This is not a defect of this file: each takes a caller-supplied `file` path whose trust is the caller's responsibility, and there is no injection, atom creation, or extraction inside these functions themselves. Out-of-tree callers are outside the reviewable codebase — not an unverified assumption about this file's behavior — so this does not become a finding here.

## Did I verify this?

- **Config eval executes attacker-supplied Elixir** — VERIFIED by execution (Elixir 1.21.0-dev / OTP 29): side-effect file written at `Config.Reader.read_imports!` time.
- **All callers pass operator-controlled paths** — VERIFIED by reading `app.config.ex`, `archive.install.ex`, `escript.install.ex`, `local/installer.ex`, plus grep-confirmed `cli.ex`, `run.ex`. `Mix.Task.run("loadconfig")` sites pass no file arg (→ `load_default`); `app.config` derives `runtime.exs` from the project's own `config_path`.
- **`CaseClauseError` on 2+ args** — VERIFIED by execution: `CRASH: CaseClauseError`.
- **No hidden dangerous sinks** — VERIFIED by full read of the file; no atom-from-bytes, external-argv command exec, archive extraction, `binary_to_term`, crypto, or network.

### Dropped in re-review

None. All prior findings (which were "no findings" plus ruled-out entries) are preserved. The two first-pass hedges are resolved rather than dropped:

- **Upgraded from hedged assumption in first pass → verified:** "I did not exhaustively audit every out-of-tree consumer / all call sites." I read the four neighbour modules that call the internal functions and confirmed all in-repo callers supply operator-controlled paths. The residual out-of-tree-caller concern is reclassified as out-of-scope (not an unverified assumption about this file), per the caveat above.
- **Upgraded from hedged assumption in first pass → verified:** "Reproductions were run against a pre-built toolchain, not the exact source tree." Both reproductions (config eval side effect; `CaseClauseError`) were re-run here against Elixir 1.21.0-dev / OTP 29, which is the version the source tree under review targets.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`persist_apps` persists reserved apps before warning (lines 99–134).** `:kernel`/`:stdlib` config is written by `put_all_env` *before* the "Cannot configure base applications" check; the check is only an advisory `Mix.shell().error`, not a security gate. Config content is operator-controlled, so no privilege boundary is crossed.
