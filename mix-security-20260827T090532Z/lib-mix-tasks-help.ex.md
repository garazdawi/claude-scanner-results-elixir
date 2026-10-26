# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/help.ex
<!-- sha256: 1cc36e24094b14e1d47aa35b9a492b03da8d87577fb4c42bd46a391cc24005c9 -->

_Analysed on 2026-08-27T11:09:50Z_

Both dependency-controlled documentation paths through `mix help` are verified to emit raw terminal control sequences end-to-end. I have what I need.

## Summary

`Mix.Tasks.Help` is a read-and-print task whose only genuinely external inputs are operator-typed argv (self-inflicted) and dependency-authored task/module documentation, which it forwards to the terminal without stripping control sequences.

## Security

#### `CSCAN-help-dep-doc-terminal-escape-injection`
- **Finding ID** — `CSCAN-help-dep-doc-terminal-escape-injection`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/mix/tasks/help.ex:295` (`build_task_doc_list/1`, `Mix.Task.shortdoc/1` → `format_task/3` → `Mix.shell().info`); also `:370` (`task_doc/1`, `Mix.Task.moduledoc/1` → `print_doc/4:206`) and `:137`–`147` (`run(["app:"…])`, `Code.fetch_docs/1` moduledoc → `IO.ANSI.Docs.print/3`).
- **Class** — Template / interpolation (terminal control-sequence injection, CWE-150)
- **Impact** — A dependency (or installed archive) ships a Mix task or module whose `@shortdoc`/`@moduledoc` contains ANSI/OSC escapes. When a developer runs `mix help`, `mix help THAT.TASK`, or `mix help app:THAT`, the escapes reach the terminal verbatim: OSC `\e]0;…\a` rewrites the window title, other OSC sequences touch the clipboard/hyperlinks on vulnerable emulators, and SGR codes can mask or forge output. The rendered content also travels to whoever *views* the output later (a pager, a CI log viewer, a reviewer's terminal) — parties who never compiled the dependency. Non-escalating against the dep itself (a loadable task implies the dep was compiled, which already runs arbitrary code), so this is defense-in-depth, but `mix help` is precisely the command used to read third-party docs one has not audited.
- **Evidence** — `verified`. Built a project with a task `Mix.Tasks.Evil` (loads via the same `Mix.Task.load_all` path a dependency's task uses) whose docs embed `\e]0;PWNED…\a`:
- `mix help` (list/shortdoc path): `od -c` shows `n o r m a l - l o o k i n g 033 ] 0 ; P W N E D - T I T L E \a summary 033 [ 3 1 m r e d 033 [ 0 m` — raw `033` (ESC) OSC + SGR bytes emitted.
- `mix help evil` (moduledoc path via `IO.ANSI.Docs.print`): `od -c` shows `033 ] 0 ; P W N E D - V I A - M O D U L E D O C \a … 033 [ 3 1 m r e d 033 [ 0 m`.
- Direct renderer check: `elixir -e 'IO.ANSI.Docs.print("Hello \e]0;PWNED-TITLE\a …", "text/markdown", …)' | od -c` likewise preserves the ESC/OSC bytes.
- **Suggested fix** — Strip or escape C0/C1 control characters (all except `\n`/`\t`) and CSI/OSC sequences from untrusted doc strings before display. The durable fix belongs downstream in `IO.ANSI.Docs.print/3` and `Mix.Shell.*info`, which render third-party content generally; as a local mitigation, `help.ex` could sanitize `Mix.Task.shortdoc/moduledoc` and `Code.fetch_docs` output before forwarding.

### Ruled out
- **`String.to_unsafe_atom(app)` (line 118) — atom-table pollution (CWE-770).** `app` is the operator-typed argv after `mix help app:`; creating one uncollectable atom per manual invocation is self-inflicted and impractical to weaponize. Confirmed the function creates a fresh atom without an existence check (`String.to_unsafe_atom("some_brand_new_atom_xyz_123")` → `:some_brand_new_atom_xyz_123`). No dependency/network path supplies this argument. (A `String.to_existing_atom/1` fallback would still be marginally safer.)
- **`Code.string_to_quoted!(string)` (line 188) — code execution / atom creation.** `string` is operator-typed argv (`t:`/`c:`/`Module` forms); it is *parsed*, not evaluated, and any atoms created are self-inflicted. Not reachable from dependency/archive/network data.
- **`apply(IEx.Introspection, type, [decomposition])` (line 192) — dynamic dispatch.** `type` is a hardcoded atom from `{:t, :b, :h}`; not attacker-influenced. `decomposition` is a decomposed AST, and the downstream handler fetches docs rather than evaluating.
- **`Mix.Project.config()[:aliases]` and `:mix`/`:iex` color config (lines 80, 181–185, 200, 230, 316–332, 363).** Sourced from `mix.exs` / `config/*.exs` / `~/.mix/config.exs`, i.e. operator-controlled code — full RCE by design, outside the attacker model.
- **`--search PATTERN` (lines 99–107).** Uses `String.contains?/2` (plain substring, no regex); no ReDoS and no pattern injection.
- **`run_iex_doc/2` `Application.put_env(:iex, :colors, …)` (lines 185/195).** Global app-env write restored in the `after` clause using previously read operator config; no external value poisons cross-tenant state.

## Risks / caveats
- The actual unsanitized-output sinks live in other files (`IO.ANSI.Docs.print/3`, `Mix.Shell`), which are outside this review; `help.ex` is the forwarder. I verified the flow through `help.ex` end-to-end against the installed `1.21.0-dev` build rather than by auditing those modules' source.
- Only the `lib/mix` subtree was present (a symlink to the source); `IEx.Introspection.decompose/2` (called at line 189) is external and unread. I judged it non-evaluating and, in any case, its input is operator-typed argv — but I did not audit it. If it were to evaluate its argument, that would still only be operator-self-inflicted via this entry point.
- The terminal-injection severity rests on a judgment that a loadable dependency task implies prior compile-time code execution (hence non-escalating → Low). If a deployment loads precompiled third-party `.beam`/archives whose *runtime* code is never executed but whose docs are browsed via `mix help`, the practical reach is somewhat higher than Low.

