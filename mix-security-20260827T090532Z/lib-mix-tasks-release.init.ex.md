# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/release.init.ex
<!-- sha256: 8bdbd58420248a3226c68b8c0fb280065c9bb151e4a0c9fc835383c0905bfb43 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/release.init.ex
<!-- sha256: 8bdbd58420248a3226c68b8c0fb280065c9bb151e4a0c9fc835383c0905bfb43 -->

_Re-analysed on 2026-08-27_

Every value that reaches a sink in this file is a compile-time constant or developer-supplied argv. The two prior hedges — "the EEx rendering happens outside this file" and "release-name validation is out of scope" — have now been resolved by reading the rendering neighbours rather than asserting; both resolve in favour of *no finding* because the only interpolated value (`release.name`) is an operator-config atom, not an attacker boundary.

## Summary
`Mix.Tasks.Release.Init` writes four fixed template files to four hard-coded `rel/*.eex` paths. No path component, filename, or file body is derived from external-attacker data. The `<%= @release.name %>` / `<%= release_mode(...) %>` EEx expressions embedded in the template *text* are rendered later by `Mix.Tasks.Release`/`Mix.Release` and interpolate only operator-controlled `mix.exs` config. There are no security findings.

## Security
No findings.

The documented public API is `run/1` (`mix release.init`). Its sole inputs are developer-supplied CLI argv, parsed with `OptionParser.parse!(strict: @switches, aliases: @aliases)` (unknown switches rejected, `release.init.ex:33`) and rejected entirely if any positional argument is present (`release.init.ex:35-37`). It then calls `create_file/3` (`release.init.ex:39-42`) with four **constant** path literals and content produced by the local text functions. No path component, filename, or file body is derived from argv, the environment, a dependency, an archive, a git ref, the network, or any file read. The `@doc false` helpers (`vm_args_text/1`, `env_text/0`, `cli_text/0`, `env_bat_text/0`, `cli_bat_text/0`) return compile-time constant strings; the only Elixir-level interpolation anywhere is `vm_args_text/1`'s `#{if(remote?, do: " -dist_listen false")}` (`release.init.ex:60`), which merges a boolean's truthiness into a fixed literal and cannot carry an attacker string.

Verified against the source-built toolchain (Elixir 1.21.0-dev / OTP 29):
- `mix release.init` in an empty dir created exactly `rel/vm.args.eex`, `rel/remote.vm.args.eex`, `rel/env.sh.eex`, `rel/env.bat.eex` — no other paths.
- `mix release.init ../../etc/passwd` → `** (Mix) Expected "mix release.init" without arguments, got: ["../../etc/passwd"]` (positional args never become paths).
- `mix release.init --evil` → rejected by strict parsing (`Unknown option`).
- `vm_args_text("; rm -rf /")` (deliberate contract violation) still emits only the fixed `-dist_listen false` suffix — no injection of the passed string.

### Ruled out

- **`vm_args_text/1` interpolation** — Confidence: high. Interpolates `if(remote?, …)`, which yields either the fixed literal `" -dist_listen false"` or `nil`; the argument's value never lands in the output. Verified by execution above. Not a finding.

- **EEx expressions `<%= @release.name %>` / `<%= release_mode(@release, "$RELEASE_MODE") %>` embedded in the generated templates** — **Upgraded from hedged assumption in first pass** (previously "outside this file / out of scope"). Confidence: high (dismissal). I traced the rendering site by reading the two neighbours:
  - The template bodies this file returns are `embed_template`'d verbatim into `Mix.Tasks.Release` (`tasks/release.ex:1510-1514`) and rendered by `copy_template(..., [release: release], force: true)` when a project supplies its own `rel/*.eex` (`tasks/release.ex:1324-1327`, `1421-1424`), or by the embedded default otherwise. The single EEx assign is `release` (`tasks/release.ex:1450-1455`).
  - `@release.name` resolves to `release.name`, which is set only from `find_release/2 → lookup_release/2 / infer_release/1` (`release.ex:170-171`, `186-246`): it is either a **key of the operator's `:releases` keyword in `mix.exs`** or `Keyword.fetch!(config, :app)` — in both cases an **atom from operator project config**. When `mix release NAME` supplies a name via argv, `lookup_release/2` raises `Unknown release` unless it matches a config-defined release key (`release.ex:227-238`), so even the argv path is constrained to operator-defined atoms.
  - `<%= release_mode(@release, "$RELEASE_MODE") %>` calls `tasks/release.ex:1500-1508`, which returns one of two fixed prefixes (`-mode $RELEASE_MODE` or `-elixir config_provider_reboot_mode $RELEASE_MODE`) selected by an operator option; `env_var` is the literal shell reference `"$RELEASE_MODE"` from this file's own template text. No external-attacker value participates.
  
  The interpolated `release.name` is therefore operator/`mix.exs`-controlled, which the folder threat model places **out of the attacker model by design** (an operator who controls `mix.exs` already has full RCE). Nothing is rendered or evaluated by *this* file. Not a finding.

- **`eval "EXPR"` / `rpc "EXPR"` and remote-shell logic in `cli_text`/`cli_bat_text`** — Confidence: high. Documented operator-facing release commands whose argument is supplied by whoever runs `bin/<release>`; the script bodies are fixed boilerplate defined as constants here. Confirmed via the neighbour read that `cli_text/0`/`cli_bat_text/0` are consumed by `Mix.Tasks.Release` (`embed_template(:cli, …)` at `tasks/release.ex:1512`, emitted through `cli_for/2` at `tasks/release.ex:1449-1456`) and are **not reachable from `run/1`** in this file. No external data reaches a sink through this file.

- **Runtime env-var handling in the generated scripts (`RELEASE_COOKIE`, `RELEASE_NODE`, `RELEASE_DISTRIBUTION`, `RELEASE_SYS_CONFIG`, …)** — Confidence: high. A property of the deployed release, not of this task's API; operator/deploy-controlled, passed in quoted argv positions. `RELEASE_DISTRIBUTION` is allow-listed to `name|sname|none` before use (`release.init.ex:143-155` for sh, `331-340` for bat), and `RELEASE_MODE` to `embedded|interactive` (`release.init.ex:342-349`). The same fixed script ships with every release; any residual concern is generic release-runtime shell review of the *deployed* artifact, not of this generator. No external-attacker boundary crosses this file.

- **`create_file/3` path handling / overwrite** — Confidence: high. The four target paths are constant literals (`release.init.ex:39-42`); `force`/`quiet` only affect the overwrite prompt. No traversal or external path component.

## Did I verify this?
- **`vm_args_text/1` non-injection** — verified by execution (source build, Elixir 1.21.0-dev / OTP 29): passing a hostile string returns only the fixed suffix.
- **`run/1` argv handling** (strict parse, positional rejection, no path derivation) — verified by execution (three CLI invocations above) and by reading `release.init.ex:32-44`.
- **Rendering-site data flow for `@release.name` / `release_mode`** — verified by reading the callgraph neighbours `tasks/release.ex` (`1324-1327`, `1421-1456`, `1500-1514`) and `release.ex` (`166-246`); `release.name` is an operator-config atom, `release_mode/2` emits a fixed prefix. This is a data-provenance trace (attacker data reaches the sink? no), which reading the neighbour fully answers; no differential/axis behaviour was involved, so no execution was required to resolve it.
- **`cli_text`/`cli_bat_text` unreachable from this file's `run/1`** — verified by reading both this file (`run/1` calls only the four `create_file` lines) and the consumer (`tasks/release.ex:1512`).

## Dropped in re-review
None. All prior dismissals are retained; the two previously hedged items ("outside this file", "out of scope for this file") are retained and **upgraded to verified dismissals** with the neighbour-module evidence cited above, rather than dropped or converted to findings — the interpolated value is operator-controlled `mix.exs` config, which is outside the folder's attacker model by design.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **EEx expressions in the generated templates (`<%= @release.name %>`, `<%= release_mode(@release, …) %>`)** — these are constant characters in template *text* written by this file; the EEx rendering and any interpolation of the release name occur later in `Mix.Release`, and the release name is operator-controlled project config (`mix.exs`), not an external-attacker boundary. Nothing is rendered or evaluated by this file.
