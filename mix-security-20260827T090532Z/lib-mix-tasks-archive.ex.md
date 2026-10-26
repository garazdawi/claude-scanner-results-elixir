# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.ex
<!-- sha256: 4b61b7569bd06980a6b0956eb4c3d730fd0981c7aadb7e38368df2203a19656e -->

_Analysed on 2026-08-27T10:22:45Z_

Both behaviors are confirmed. Key findings from the reproduction:

1. **Terminal-injection is behaviorally real but not a boundary crossing.** `Mix.Shell.IO.info/1` calls `IO.puts(IO.ANSI.format(["* ", item]))`. `IO.ANSI.format` only interprets *atom* codes; raw ESC(27)/BEL(7) bytes inside a binary filename pass through untouched (verified: output retained bytes 27 and 7, and `cat -v` showed `* evil-^[[31mHACK^[[0m`). But the only way a hostile-named entry lands in `~/.mix/archives` is `archive.install` — and an installed archive is loaded as a plugin (`Mix.Local.append_archives` → `Code.append_path`) that runs arbitrary code on *every* subsequent `mix` command. So the precondition for a malicious filename already grants full RCE; escape-injection-on-listing is no privilege gain.

2. **`Path.wildcard` / `MIX_ARCHIVES`** is `System.get_env("MIX_ARCHIVES") || mix_home()` — operator-controlled config, explicitly out of scope.

The file has no atom creation, code eval, command execution, deserialization, network, or file-write sinks. `run(_)` ignores argv.

## Summary
`Mix.Tasks.Archive` is a 15-line read-only listing task whose only sinks (`Path.wildcard` over the operator's `MIX_ARCHIVES` dir, and printing entry basenames) sit entirely inside the operator/local-trust boundary, so no attacker-reachable vulnerability is present.

## Security

No findings

### Ruled out
- **Terminal-escape / control-char injection via printed filenames** (`print/1`, line 37 — `Mix.shell().info(["* ", item])`). Verified that the print path (`IO.puts(IO.ANSI.format(...))`) passes raw ESC/BEL bytes in a filename straight to the terminal (`elixir` repro output kept bytes `27` and `7`; `cat -v` rendered `* evil-^[[31mHACK^[[0m`). Not a finding: the only supported way to place an entry in the listed directory is `mix archive.install`, and an installed archive is auto-loaded as a code-path plugin that executes arbitrary code on any later `mix` invocation. The precondition already yields full RCE by design (`archive.ex` `@moduledoc` and the folder threat model both state archives must come only from trusted sources), so spoofing list output is strictly dominated and gains the attacker nothing. Any other path to a hostile filename requires local write access to `~/.mix/archives`, which the threat model puts out of scope.
- **`Path.wildcard(Mix.path_for(:archives) <> "/*")`** (`run/1`, lines 25–27). The base directory is `System.get_env("MIX_ARCHIVES") || Mix.Utils.mix_home()/archives` (`mix.ex:658`) — operator-supplied config, not attacker-influenced. Glob-metacharacter or wildcard-cost concerns reduce to the operator affecting their own environment, which is out of scope. No confinement issue because nothing is written; entries are only read and their basenames printed.
- **Argument injection into `run/1`** — the task ignores its argument (`run(_)`), so no attacker-controlled argv reaches any sink.

## Risks / caveats
- The behavioral reproductions used the sandbox's prebuilt `/opt/elixir` because the source tree under review ships only `lib/` (no compiled BEAM); the `archive.ex` pipeline and the `Mix.Shell.IO.info`/`IO.ANSI.format` code paths exercised are byte-identical to the source under review, so this does not affect the conclusions.
- Whether `mix archive.install` sanitizes control characters out of the stored entry name lives in a *different* file (`archive.install.ex`) and was not audited here; even if it does not, the RCE-domination argument above makes it immaterial to `archive.ex`. That sanitization question belongs to a review of the install task, not this listing task.

