# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/escript.build.ex
<!-- sha256: c20c28f2745a4ee55780cebb510dd43b3685fc82842984cc06d39b5025792d08 -->

_Analysed on 2026-08-27T11:00:19Z_

I've completed the trace. Everything dangerous in this file is either operator-controlled (`mix.exs`/config, explicitly out of scope) or reachable only by a malicious dependency that already obtained compile-time code execution via the `Mix.Task.run("compile", args)` on line 141 — which runs *before* any dep-influenced code in this task. No sink here crosses a trust boundary the attacker doesn't already command. Here is the report.

## Summary
`mix escript.build` packages the operator's own project, config, and dependencies into an escript; every security-relevant input is either operator-controlled (out of scope by design) or reachable only by a dependency that has already executed arbitrary code during the `compile` this task runs first, so no sink crosses a new trust boundary.

## Security

No findings

### Ruled out

- **`String.to_unsafe_atom/1` (line 191, `escriptize`)** — inlines to `:erlang.binary_to_atom` (creates an atom, no existing-atom check). Argument is `Atom.to_string(app) <> "_escript"`, where `app = Keyword.get(escript_opts, :app, project[:app])` comes from the operator's `mix.exs`. One bounded atom per build from an operator-owned name — not atom-table exhaustion and not attacker-derived. (Safe form would be `String.to_existing_atom/1`, but it is unnecessary here since the module is then created via `Module.create/3` and must be new.)
- **`Config.Reader.read!/2` (line 368) and embedded `Config.Reader.eval!/3` (line 416)** — evaluate `config/config.exs` and `config/runtime.exs`. These are the operator's own project files (`project[:config_path]`), executed as code by design; the escript docs explicitly state both configs are included. Operator boundary, not attacker.
- **`Module.create/3` (line 407)** — builds a module from quoted AST assembled entirely from operator-controlled values (`module`, `app`, `compile_config`, `runtime_config`, `language`). No externally-derived AST is `unquote`d.
- **`File.write!/2`, `File.mkdir_p!/1`, `File.chmod/2` (lines 214–216, 249)** — target is `filename = escript_opts[:path] || name_for(:escripts, project)`, i.e. the operator's `:path`/`:name`/app name. Operator-controlled output path; no external component.
- **`Path.wildcard/2` (lines 231, 235, 342)** — patterns are built from `Mix.Project.app_path()`, `dep.opts[:build]`, and `Mix.Project.consolidation_path/1` (build directories), not from attacker-controlled glob text. Cost is bounded by the project's own build tree.
- **`File.read!/1` (lines 304, 379) / `:zip.create/3` (line 207)** — reads compiled `.beam`/`.app`/priv files from dep build dirs and *creates* an in-memory zip. This is archive **creation**, not extraction, so there is no zip-slip write on the build host; entry names (`Path.relative_to(path, apps_dir)`) are stored, not resolved to filesystem paths here.
- **`unquote(module).main(args)` dispatch (lines 437, 450)** — `module` is `escript_opts[:main_module]` from `mix.exs`, and is gated by `Code.ensure_loaded?/1` (line 163). Operator-controlled dispatch target.
- **`build_comment/1`, `build_emu_args/2` (lines 355–361)** — interpolate `escript_opts[:comment]`/`:emu_args` (and the operator-derived `escript_mod`) into the escript header. Newline/metacharacter injection here is operator-into-their-own-escript; self-inflicted, no boundary crossed.
- **`extra_apps_in_app_tree/1` app-atom handling (lines 271–293)** — recurses over `:applications`/`:included_applications` from `.app` specs; the app names are already-existing atoms (loaded by OTP's `.app` consult), and `Application.load`/`Application.spec` take atoms, so no external-string→atom conversion occurs here.

## Risks / caveats

- **Unbounded recursion on a cyclic `.app` (`extra_apps_in_app_tree/1`, lines 280–292)** — there is no cycle guard; `Enum.uniq/1` is applied only *after* the recursion returns (line 268). A dependency shipping a hand-crafted `ebin/<app>.app` whose `:applications` forms a cycle (e.g. lists itself) would recurse forever, hanging/crashing the build when `embed_elixir` is true (the Elixir default). I did **not** report this as a finding: the same dependency already runs arbitrary code during the `mix compile` invoked on line 141 (before this code), so it crosses no new trust boundary and this codebase treats a compile-time-RCE actor merely stalling the build as uninteresting. I reasoned about this rather than staging a full crafted-dep build (**asserted**).
- **Dep-priv symlink exfil into the distributed escript** — with `:include_priv_for` set for a dependency, `get_files/2` collects `priv/**/*` via `Path.wildcard` filtered by `File.regular?/1` (which follows symlinks), then `read_beams/1` reads each with `File.read!`. A malicious dep could, in principle, place a symlink under its `priv/` pointing at a host file so its contents get embedded in the escript the operator then distributes. This requires operator opt-in (`:include_priv_for`) and, again, the dep already has compile-time RCE, so it is not a privilege escalation. I did not empirically confirm whether `Path.wildcard`'s `**` traverses symlinked directories on this build (**asserted**); it does not change the conclusion.
- **Scope note** — I reviewed only this file. The safety of `Mix.Release.strip_beam/2` (line 319) on malformed dep `.beam` input, and of any later `:escript.extract/2` that unpacks the produced archive, lives in other files and was not assessed.
- The MCP servers `claude.ai`/Google Drive require authorization and were unavailable; this did not affect the review, which needed only the local source and the from-source Elixir/OTP-29 toolchain.

