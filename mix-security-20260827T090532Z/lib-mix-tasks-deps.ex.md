# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.ex
<!-- sha256: 6ba99102f68809933765c4dde8ca9dfc7a2d9e61c096cb514c06252b9410459c -->

_Analysed on 2026-08-27T10:47:18Z_

I have completed my per-sink analysis. Let me record what I've established before writing the report.

The file is a thin, read-only listing task (`mix deps`). I traced every sink:

- **`String.to_unsafe_atom/1` (line 209)** — confirmed it's `:erlang.binary_to_atom(s, :utf8)`, creates non-GC'd atoms with no validation. But its input is `apps` = the *positional command-line arguments* the developer types into `mix deps <names>`. Verified the design intent: `to_existing_atom` **raises** on an unknown name, so the creating variant is deliberately chosen to report "unknown dependency" gracefully (reproduced: `mix deps totally_bogus_dep_name_12345` → `warning: unknown dependency ...`).
- **`OptionParser.parse` (line 202, non-strict)** — verified it *also* creates atoms for arbitrary unknown `--flags`. Same CLI/developer trust boundary; lives in another file.
- **`scm.format_lock(dep.opts)` (line 229)** — dispatch on the dep's resolved SCM module (constrained to registered SCMs), a benign formatting callback.
- **`shell.info`/`shell.error` prints (227/230/233/238)** — the interpolated strings (`format_dep`, `format_status`, `format_lock`) are built in `Mix.Dep`/`Mix.SCM.*`, not here; this file adds no unescaped attacker bytes.

None of these cross the DEPENDENCY / archive / git-ref / network / read-file boundary *within this file*.

## Summary

`Mix.Tasks.Deps` is a thin, read-only listing task whose only external inputs are the developer's own `mix deps` command-line arguments; no dependency-, archive-, git-, or network-derived data reaches a dangerous sink inside this file, so there are no findings attributable to it.

## Security

No findings.

### Ruled out

- **`String.to_unsafe_atom/1` on `apps` (line 209).** This is the deliberately-named atom-*creating* variant (verified: `:erlang.binary_to_atom(s, :utf8)`, no validation, atoms never GC'd — CWE-770 shape). Its input is the positional CLI arguments to `mix deps <app…>` — the developer's live keystrokes, not data from a dependency, archive, git ref, network peer, or a file the project reads. That is the "developer trusts their own input" case the scope explicitly excludes. It is also intentional and justified: `String.to_existing_atom/1` raises `ArgumentError` on an unknown name (verified), whereas the creating variant lets an unknown/typo'd name flow through to a graceful `warning: unknown dependency` (verified by running `mix deps totally_bogus_dep_name_12345`). The process is short-lived and args are OS `ARG_MAX`-bounded, so even self-inflicted atom exhaustion is impractical.
- **Atom creation by non-strict `OptionParser.parse` (line 202).** Verified that `OptionParser.parse(args, switches: [all: :boolean])` creates atoms for arbitrary unknown `--flags`. Same CLI/developer trust boundary as above, and the behavior lives in `OptionParser`, not this file.
- **`scm.format_lock(dep.opts)` behaviour dispatch (line 229).** `scm` is the SCM module resolved by the converger against the registered SCM set (`Mix.SCM.available`), not an atom derived from untrusted bytes; registering a custom SCM already requires build-time code execution (a `mix.exs`), and `format_lock/1` is a benign string formatter. No arbitrary-module or atom-injection escalation is introduced here.
- **Terminal/log injection at `shell.info`/`shell.error` (lines 227, 230, 233, 238).** The interpolated values come from `format_dep/1`, `format_status/1` (both `Mix.Dep`), and `scm.format_lock/1` (`Mix.SCM.*`). This file only splices already-constructed strings via `#{}` (Elixir interpolation, not an Erlang `~s` format string) and introduces no raw attacker bytes of its own; any escaping obligation for dependency-derived content (e.g. a transitive dep's `:branch`/`:tag` name) belongs to those constructor functions, not to this printer.

## Risks / caveats

- **Cross-file terminal-escape exposure (not this file's bug).** A malicious transitive dependency could declare data such as `app:`/`:branch`/`:tag` containing ANSI escape sequences; those would flow through `Mix.Dep.format_dep/1`, `Mix.Dep.format_status/1`, or `Mix.SCM.Git.format_lock/1` and be emitted verbatim by the `shell.info` calls here. Whether that is sanitized must be judged in `lib/mix/dep.ex` and the SCM modules — outside the file under review. Flagging so a folder-level pass can confirm those constructors neutralize control characters.
- **Limited checkout.** Only the `lib/mix` subtree is present in this checkout (Elixir stdlib is the installed `/opt/elixir` build, OTP 29). I confirmed `String.to_unsafe_atom`, `to_existing_atom`, and `OptionParser` behavior against that installed build, but could not read the `Mix.Dep`/`Mix.SCM` source from this working directory to fully audit the printed-string constructors mentioned above.

