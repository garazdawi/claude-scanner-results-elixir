# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/cmd.ex
<!-- sha256: 1a5ab0471bb55ca78404397ee19b809c25ba8e9cff36ab281c7ec4788a683498 -->

_Analysed on 2026-08-27T10:28:29Z_

## Summary

`Mix.Tasks.Cmd` executes an operator-supplied command line; every sink (atom creation from `--app`, `Path.expand` on the command path, and shell/argv dispatch to `Mix.shell().cmd`) is fed only by argv or a `mix.exs` alias, so no dependency/archive/network/file trust boundary is crossed and no security finding is warranted.

## Security

No findings

### Ruled out

- **Atom creation via `String.to_unsafe_atom/1` (line 109).** This is the Elixir 1.21 rename of the now-deprecated `String.to_atom/1` — same dynamic, non-GC'd atom creation (verified: 2000 calls added 2187 permanent atoms). It is applied to `--app` values obtained from `OptionParser.parse_head!` on the process argv (or a `mix.exs` alias the operator wrote). That data is never forwarded from a dependency, archive, git ref, network peer, or a file the project merely reads — the only internal callers grep turned up are none; `mix cmd` is reached solely from operator-controlled invocations. Atom-table exhaustion here would be self-inflicted (the operator typing thousands of distinct `--app` flags), which the review scope explicitly treats as uninteresting. Defense-in-depth would be `String.to_existing_atom/1`, since the compared `Mix.Project.config()[:app]` atom already exists, but this is not a security defect given the trust boundary.
- **Shell command injection via `--shell` (lines 121, 133 → `Mix.Shell.cmd/3` → `System.shell`).** When `--shell` is passed, `Enum.join([path | rest], " ")` is run through `System.shell`, so metacharacters are interpreted (verified: `mix cmd --shell echo 'hi && touch PWNED_SHELL'` created the file). This is the documented purpose of the flag ("perform shell expansion of the arguments"), and the command string is the operator's own input — by design, not a vulnerability.
- **Command/option injection on the default (argv) path (lines 124–128, 133 → `System.cmd`).** Without `--shell`, `arg` is a `{path, rest}` tuple routed to `System.cmd`, which passes an explicit argv and does not invoke a shell (verified: `mix cmd echo 'hi; touch PWNED_DEFAULT'` printed the literal string and created no file). Values are operator-supplied; a value starting with `-` could in principle option-inject into the target program, but the target and its arguments are entirely operator-chosen, so no external actor influences them.
- **Path traversal via `Path.expand(path, cd)` (line 125).** Both the command `path` and the `--cd` directory are operator-supplied; expansion only resolves the executable to launch, and nothing confines or is meant to confine it to a root.

## Risks / caveats

- The full Elixir standard library is not in this checkout (`elixir/lib` contains only a `mix` symlink), so I could not read the `String.to_unsafe_atom/1` and `Path.expand/2` sources directly; I characterized them by fetching their in-VM docs and by running the built-from-source toolchain (Elixir 1.21.0-dev / OTP 29) instead.
- My "operator-only" conclusion rests on the absence of any internal caller forwarding untrusted data into `mix cmd` (grep found none) and on the folder threat model's stance that a dependency's `mix.exs`/build scripts already run arbitrary code by design — so even if a malicious dependency's build invoked `cmd --shell`, that adds no attack surface beyond what the dependency already has. If a downstream tool were built that feeds *remote-derived* strings into `Mix.Task.run("cmd", ...)`, the `--shell`/`--app` sinks would become live; that is a property of the caller, not of this file.

