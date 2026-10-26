# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/release.init.ex
<!-- sha256: 8bdbd58420248a3226c68b8c0fb280065c9bb151e4a0c9fc835383c0905bfb43 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/release.init.ex
<!-- sha256: 8bdbd58420248a3226c68b8c0fb280065c9bb151e4a0c9fc835383c0905bfb43 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`Mix.Tasks.Release.Init` is a Mix task whose only real logic is `run/1` (parse two boolean switches, reject extra args, write four static template files via `Mix.Generator.create_file/3`); the rest is embedded shell/batch/EEx template text returned by `@doc false` helper functions. I found no genuine, reachable Elixir defect through the documented public API.

This re-review verified each hedged assumption from the first pass by (a) reading the callgraph neighbours (`Mix.Generator` in `generator.ex`, `Mix.Tasks.Release` in `release.ex`) and (b) executing the one dynamic interpolation on a live Elixir toolchain. Nothing needed upgrading to a low-confidence finding; the hedges resolved to verified facts.

## Bugfixes

No findings

## Risks / caveats

- The bulk of the file is static text returned by `@doc false` helpers (`cli_text/0`, `cli_bat_text/0`, `env_text/0`, `env_bat_text/0`, `vm_args_text/1`). I confirmed by grep that all five are consumed at **compile time** by `Mix.Tasks.Release` in `release.ex:1510–1514` via `embed_template(...)`; `cli_text`/`cli_bat_text` are *not* reached by this module's `run/1` at all. Defects internal to the generated `.sh`/`.bat` scripts are shell/batch semantics, not Elixir program behavior of this module, and are not reachable as an Elixir defect through `run/1`.
- The only dynamic Elixir logic is the `if(remote?, ...)` interpolation in `vm_args_text/1`; I executed it (see "Did I verify this") and it is correct for `true`/`false`/`nil`/truthy.

### Ruled out

- **`run/1` option handling** — `OptionParser.parse!/2` with `strict: @switches` and `aliases: @aliases` raises on unknown switches (documented task behavior); `-f` aliases to `:force`. Both parsed switches (`:force`, `:quiet`) are consumed by the sink. **Confidence: high.**
  - Axes examined: option-key validity (which keys `create_file/3` reads), unknown-switch handling, alias resolution. Not examined: none — the strict switch set is exactly `[:force, :quiet]`, and I read the sink (`generator.ex:36–53`, `log/4` at `:175`) confirming `opts[:force]` gates overwrite and `opts[:quiet]` gates logging; there are no other option keys that could be silently dropped.

- **`opts` pass-through to `Mix.Generator.create_file/3`** — Verified against the neighbour: `create_file/3` (`generator.ex:36`) reads `opts[:force]` (line 39) and passes `opts` to `log/4` which reads `opts[:quiet]` (line 175). Both `:force` and `:quiet` are documented options (`generator.ex:24–25`). No third option key is required or silently ignored. **Confidence: high.**
  - Axes examined: option-key set consumed by sink, `@spec` shape (`create_file(Path.t(), iodata, generator_opts) :: boolean()`). Not examined: none — the two switches produced by `run/1` are precisely the two the sink documents and reads.

- **`vm_args_text/1` interpolation** — executed the exact interpolation (`"...6789#{if(remote?, do: " -dist_listen false")}"`) on the live toolchain for `false`, `true`, `nil`, and a truthy string. `if` returning `nil` interpolates to `""`; truthy appends `" -dist_listen false"`. No crash, no truthiness/precedence bug. **Confidence: high.**
  - Axes examined: `remote? = false`, `true`, `nil`, truthy non-boolean. Not examined: none — all reachable call sites pass a literal `false` (`vm_args_text(false)` at `release.init.ex:39` and `release.ex:1510`) or literal `true` (`release.init.ex:40`), both of which I ran.

- **`@moduledoc` vs. code mismatch** — the four files the moduledoc claims to create (`rel/vm.args.eex`, `rel/remote.vm.args.eex`, `rel/env.sh.eex`, `rel/env.bat.eex`) match the four `create_file/3` calls (lines 39–42) exactly; no documentation-contract discrepancy. **Confidence: high.**
  - Axes examined: filename list, ordering, count. Not examined: none — direct textual comparison of moduledoc lines 14–17 against calls 39–42.

- **Embedded shell/batch script bugs** (e.g. `:eval` `set args=%args%` handling, word-splitting on `$(release_distribution ...)`) — these live in `@doc false` template helpers. I confirmed they are consumed only by `release.ex:1510–1514`'s `embed_template`, not by this module's `run/1`, and they are shell/bat runtime semantics rather than Elixir program behavior. Out of scope for this file's Elixir defect surface. **Confidence: high** (that they are out of scope as *Elixir* defects reachable via `run/1`).
  - Axes examined: reachability from `run/1` (none), consumer identity (compile-time `embed_template` in `release.ex`). Not examined: the runtime behavior of the emitted shell/batch scripts themselves — intentionally, as that is not Elixir behavior of this module and belongs to a shell-script review, not this file's per-file Elixir review.

### Did I verify this

- **`Mix.Generator.create_file/3` consumes `:force`/`:quiet`** — verified by reading `generator.ex:24–53` and `:175` (the `log/4` `:quiet` gate). Both switches flow soundly. **Upgraded from hedged assumption in first pass** ("both parsed switches ... are valid options for `Mix.Generator.create_file/3`" is now read-confirmed, not assumed).
- **Helper texts consumed elsewhere, not by `run/1`** — verified by grep: `release.ex:1510–1514` embeds all five helpers via `embed_template`. The first-pass hedge "they are consumed by `Mix.Release` elsewhere" is now precisely located (`Mix.Tasks.Release` in `release.ex`), read-confirmed. **Upgraded from hedged assumption in first pass.**
- **`vm_args_text/1` interpolation** — verified by executing it under `/opt/elixir/bin/elixir`; output for `false`/`nil` is the string without the flag, for `true`/truthy it appends `" -dist_listen false"`. Confirmed no crash and correct output.
- **Not verified at runtime:** the behavior of the generated `.sh`/`.bat` scripts. This is deliberate and out of scope — they are not Elixir program behavior and are not reachable as an Elixir defect through the documented `run/1`.

### Dropped in re-review

None. Every prior ruled-out entry is preserved; the two that rested on cross-file assumptions (`create_file/3` option validity, and helper-text consumers) were upgraded from asserted to read-verified rather than dropped.
