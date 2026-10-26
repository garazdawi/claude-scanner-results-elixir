# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/app.start.ex
<!-- sha256: 3adc6c01fe83737119901cbd31ab081ad8a3bd76d925b5ed7cc569b9ae13e491 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/app.start.ex
<!-- sha256: 3adc6c01fe83737119901cbd31ab081ad8a3bd76d925b5ed7cc569b9ae13e491 -->

_Re-analysed on 2026-08-27_

I re-read the entire file and, for this pass, read the two callgraph neighbours that the first-pass report had only assumed about: the error/print sink (`Mix.Shell.IO.error/1` in `lib/mix/shell/io.ex`) and the umbrella app-name source (`Mix.Dep.Umbrella` in `lib/mix/dep/umbrella.ex`). I also re-ran the interpolation/format flow through a live Elixir to confirm no format-directive re-interpretation occurs.

## Summary

`Mix.Tasks.App.Start` is a thin orchestration task whose only inputs (command-line `args` and `Mix.Project.config()`) are developer/operator-controlled. Its one string-building sink (`could_not_start/2`) reaches only `Mix.shell().error/1` / `Mix.raise/1`, both of which I have now confirmed treat the string as literal terminal output, not as a format string or downstream interpreter. No sink here is reachable from a dependency, archive, git ref, or network peer — no findings.

## Security

No findings

### Ruled out

- **`could_not_start/2` interpolation → `Mix.shell().error/1` (lines 84, 89, 103–105) — "template/interpolation" / format-string injection.**
  `could_not_start(app, reason)` builds a plain binary: `"Could not start application #{app}: " <> Application.format_error(reason)`. That binary is passed as an iolist element to `Mix.shell().error(["** (Mix) ", ...])` or to `Mix.raise/1`.
  I read the default shell sink `Mix.Shell.IO.error/1` (`shell/io.ex:42–45`): it calls `IO.puts(:stderr, IO.ANSI.format(red(message)))`, where `red/1` wraps the message as `[:red, :bright, message]`. `IO.ANSI.format` interprets only the literal atom elements (`:red`, `:bright`) as escape sequences; the message string is emitted verbatim. There is no `~s`/`~p`/`io_lib:format` step, no shell, and no query — nothing re-interprets the interpolated content.
  Executed confirmation (this pass): interpolating an atom yields only its name (`"...my\e[31mapp: {:bad, :thing}"`), and `IO.ANSI.format([:red,:bright, ["** (Mix) ", msg]])` passes the message bytes through unchanged with no directive expansion. An atom containing raw ANSI escapes would reach the terminal verbatim, but see the umbrella entry below for why that value is not attacker-controlled.
  Confidence: high. `Not examined: none` — the sink module was read and the full flow executed.

- **Source of `app` / `apps` — `apps(config)` and `Mix.Dep.Umbrella.cached/0` (lines 95–101).**
  `apps` is `[config[:app]]` (the project's own app atom) or, for umbrellas, `Enum.map(Mix.Dep.Umbrella.cached(), & &1.app)`. I read `Mix.Dep.Umbrella`: `cached/0` → `loaded/0` → `unloaded/0`, whose app atoms come from `Mix.Project.apps_paths(config)` — i.e. the developer's own in-repo `apps/*` sub-project directories (each carried as `%Mix.Dep{scm: Mix.SCM.Path, app: app}`). These are local operator-authored sub-projects, not fetched dependencies, archives, or network-supplied names. Passing them to `Application.ensure_all_started/2` and interpolating them into the error line is therefore self-inflicted, outside the attacker model.
  Confidence: high. `Not examined: none` — the source module was read to ground provenance.

- **`Application.ensure_all_started(apps, ...)` (line 75) — "code execution".** Starting an application runs its `Application.start/2` callbacks (arbitrary code), but `apps` is the project's own app or its local umbrella children (confirmed above). Running your own/declared apps' start callbacks is the entire purpose of `mix app.start` and is trusted-by-design per the folder threat model. No attacker-controlled module/app name is injected.
  Confidence: high.

- **`OptionParser.parse(args, switches: @switches)` and `"--no-start" in args` (lines 54, 56) — argument handling.** `args` are the command-line arguments the developer typed (or a parent Mix task forwarded); operator-controlled, not a trust boundary. `@switches` is a fixed literal.
  Confidence: high.

- **`type/2` and the `when type == :permanent` guard in `start/3` (lines 79, 108).** `type/2` returns only `:temporary` or `:permanent`, so both `start/3` error clauses are reachable — not dead defensive code. `type/2` and `start/3` are `@doc false`/internal; reaching them with hostile arguments requires calling non-public helpers directly, which is out of scope.
  Confidence: high.

- **`System.stop(1)` + `Process.sleep(:infinity)` (lines 85–86) and `Logger.App.stop()` (line 63) — shared/global state & concurrency.** Intentional lifecycle actions (stop node on permanent-app failure; stop Logger before app start). The acknowledged race in the permanent branch is a robustness note; no external input steers whether or when they fire.
  Confidence: high.

- **`config = Mix.Project.config()` / `config[:start_concurrently]` / `config[:app]` (lines 64–66, 98) — config as input.** These come from the developer's `mix.exs`, which is executed as code and trusted-by-design (an operator controlling it already has full RCE). Not a boundary crossing.
  Confidence: high.

## Risks / caveats

- The file still delegates real work to `Mix.Task.run("app.config", args)`, `Application.ensure_all_started/2`, and `Logger.App.stop/0`. Any vulnerability in those lives in other files (`app.config.ex`, OTP's application controller, Logger); nothing in this file weakens or misuses them. I did not audit those modules — but no value in *this* file reaches an unverified sink, so the delegation is a scoping note, not an unverified safety assumption about this file.

## Did I verify this?

- **Format-string / injection at the error sink:** Verified by reading `Mix.Shell.IO.error/1` (`shell/io.ex:42–45`, plus `red/1` at 112–114) and by executing the `could_not_start` → `IO.ANSI.format` flow with an exotic ANSI-bearing atom. The message is emitted literally; no `~s`/`~p` re-interpretation exists. No longer an assumption.
- **Provenance of `app`/`apps`:** Verified by reading `Mix.Dep.Umbrella` (`dep/umbrella.ex`), confirming app atoms originate from local `apps_paths` sub-projects (`Mix.SCM.Path`), i.e. developer-owned in-repo code. No longer an assumption.
- **Downstream modules (`app.config`, application controller, Logger):** Not read — out of scope for this file, and no in-file value flows to a sink that depends on their internals. Stated as a caveat, not relied on for any ruled-out claim.

## Dropped in re-review

None. Every prior ruled-out entry is preserved; the two first-pass hedges ("out of scope for this single-file review" regarding the error sink and umbrella app names) have been **upgraded from hedged assumptions to verified ruled-out entries** by reading `shell/io.ex` and `dep/umbrella.ex` and executing the format flow. No prior finding existed to drop.
