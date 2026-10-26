# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell/io.ex
<!-- sha256: fee8abdbdf0b7dd2a1d6658cb7236a1cfef441a0ab24ecc0d9006b5a3038c7b7 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell/io.ex
<!-- sha256: fee8abdbdf0b7dd2a1d6658cb7236a1cfef441a0ab24ecc0d9006b5a3038c7b7 -->

_Re-analysed on 2026-08-27_

This re-review discharges the three hedged assumptions in the first pass. I read the two neighbour modules the hedges referred to — `Mix.Shell` (`shell.ex`) and `Mix.ProjectStack` (`project_stack.ex`) — and executed the passthrough / trust-direction axes on a working OTP 29 + Elixir 1.21-dev toolchain. Nothing upgraded to a finding; every hedge resolved to a verified ruled-out entry.

## Summary

`Mix.Shell.IO` is a thin, faithful wrapper over `IO.puts`/`IO.gets`/`IO.write` and `Mix.Shell.cmd/3`. It introduces no parser, decoder, path, atom-creation, or code-execution sink of its own, and its one trust-decision function (`yes?/2`) fails safe in the documented "dangerous operation" mode. No security vulnerability is reachable through its documented public API.

## Security

No findings.

### Ruled out

- **Terminal-escape / control-character passthrough in `info/1`, `error/1`** (`io.ex:34`, `42`).
  Executed: `IO.ANSI.format(["hello\e[2J\aworld"], true)` and `... false)` both return the input **byte-for-byte** (`ansi-on passthrough identical: true`, `ansi-off passthrough identical: true`). So any untrusted substring a caller folds into a message reaches the terminal verbatim, ANSI enabled or not. This is genuine Mix-wide exposure but **not a defect localized to this file**: the `Mix.Shell` behaviour types the argument as `IO.ANSI.ansidata()` (`shell.ex:39`, `:44` — read this pass), i.e. the caller is explicitly asking the shell to emit formatting/control data. Sanitizing here would break the module's documented purpose (colored output). The trust boundary is crossed in whichever caller interpolates untrusted bytes; this is the trusted output primitive. Reportable against callers, not `io.ex`.
  Confidence: High.
  Axes examined: ANSI-enabled formatting, ANSI-disabled formatting, embedded CSI (`\e[2J`) and BEL (`\a`). | Not examined: none.

- **Command execution in `cmd/2`** (`io.ex:120`–`127`).
  Verified by reading `shell.ex:145`–`168` (this pass): `Mix.Shell.cmd/3` runs a **binary** `command` through `System.shell(command, options)` (`shell.ex:164`) — a real shell-injection sink — while a `{executable, args}` tuple goes through the safe `System.cmd/3` (`shell.ex:163`). Confirmed in `io.ex`: `cmd/2` forwards the caller's `command` **verbatim** to `Mix.Shell.cmd/3` and constructs nothing from network/file/dependency input of its own (`io.ex:123`). The sink and any string-building both live outside this file. Not this file's finding.
  Confidence: High.
  Axes examined: binary-command path, tuple-command path, in-file string construction (none). | Not examined: none.

- **`print_app/0` banner emits an unsanitized app name** (`io.ex:23`–`29`).
  *Upgraded-in-scope from a first-pass "Risks / caveats" hedge* ("I did not verify whether Mix rejects control characters in a dependency-supplied `:app`"). Resolved this pass:
  (1) Read `Mix.Shell.printable_app_name/0` (`shell.ex:114`–`117`) → delegates to `Mix.ProjectStack.printable_app_name/0` (`project_stack.ex:221`–`234`), which returns `h.config[:app]` — the raw project-config atom, with **no** control-character validation anywhere in the chain.
  (2) Executed: `"==> #{:"evil\e[2Jname"}"` yields `"==> evil\e[2Jname"` (`app-name escape passthrough: true`) — a control-char atom renders verbatim in the banner.
  So the escape path is real. It is nonetheless not a defect localized to `io.ex`: the `:app` atom originates in `mix.exs`/config, which the threat model classes as operator-trusted (RCE by design). A malicious *dependency* controls its own `:app`, but such a dependency already executes arbitrary code during `deps.compile` (by design), so terminal-escape injection via the banner is strictly weaker than the RCE it already holds — no new capability. `io.ex` is the output primitive; the untrusted atom is admitted upstream in config loading, which also does not sanitize. Attributable to config/ProjectStack, not here.
  Confidence: High (that the passthrough is real and unsanitized; that it grants no capability beyond the dependency's existing RCE).
  Axes examined: atom→banner interpolation with embedded CSI; source of the atom (`config[:app]`, unsanitized); trust origin (operator/dependency, both already-RCE). | Not examined: none.

- **`yes?/2` trust decision** (`io.ex:88`–`110`).
  Executed the trim/accept logic directly: `default: :no` empty line → `false` (`no-default empty -> should be false: false`); `default: :yes` empty line → `true` (`yes-default empty -> true: true`); `is_binary(:eof)` → `false`, so EOF falls through the catch-all (`io.ex:108`) and rejects regardless of default. `default` is validated against `[:yes, :no]` with a raise (`io.ex:91`–`94`). No wrong-way-safe path: the documented dangerous-operation mode (`default: :no`) never returns `true` without genuine affirmative input.
  Confidence: High.
  Axes examined: empty-line under each default, EOF (`:eof` non-binary), bad-default raise, affirmative-answer set. | Not examined: none.

- **`prompt/1` / `IO.gets` error handling** (`io.ex:52`–`59`).
  Returns raw input by contract; raises a clear message on an `{:error, _}` device (`io.ex:56`). No sink; nothing externally influenced beyond the interactive user's own keystrokes.
  Confidence: High.
  Axes examined: error-tuple branch, normal-answer branch. | Not examined: none.

- **Dead/defensive code.** All guards reachable: the `default not in [:yes, :no]` raise fires on a bad option; both `{:error, _}` clauses fire on a closed/stderr device. None unreachable.
  Confidence: High.
  Axes examined: reachability of each guard clause. | Not examined: none.

## Risks / caveats

- The terminal-escape passthrough is genuine and worth fixing *at the callers* that mix untrusted data (dependency names during `mix deps.compile`, compiler diagnostics carrying dependency-controlled file paths, Hex/git strings) into `Mix.shell().info/error`. Those call sites live in other files and are out of scope for a per-file review of `io.ex`; I did not enumerate them. If a systemic fix is desired, the natural place is a shared scrubber applied by callers before formatting, not inside this trusted output module.
- `Mix.Shell.cmd/3`'s `System.shell/2` on a binary `command` is a real injection sink (`shell.ex:164`). Whether any internal caller of `Mix.Shell.IO.cmd/2` builds `command` from dependency/network input is an audit that belongs to `shell.ex` and the task modules that call `cmd`, not to this file — `io.ex` itself constructs nothing.

## Did I verify this?

- **`IO.ANSI.format/1` control-byte passthrough** — Verified by execution (both ANSI-enabled and disabled return input verbatim).
- **`cmd/2` forwards verbatim to a `System.shell` sink that lives in `shell.ex`** — Verified by reading `shell.ex:145`–`168` and confirming `io.ex:123` constructs nothing.
- **`print_app/0` app name is `config[:app]`, unsanitized, and control chars survive** — Verified by reading `shell.ex:114`–`117` + `project_stack.ex:221`–`234` (no sanitization) and by executing the atom-interpolation passthrough.
- **`yes?/2` fail-safe direction and EOF rejection** — Verified by execution of the trim/accept predicate and `is_binary(:eof)`.

## Dropped in re-review

None. Every ruled-out entry from the first pass is preserved; the three first-pass hedges ("did not audit call sites", "did not trace every internal caller", "did not verify whether Mix rejects control characters in `:app`") were converted from asserted to verified by reading the two neighbour modules and running the relevant axes. No prior finding existed to drop, and no dismissal was upgraded to a finding — all three hedges resolved to real-but-out-of-file passthroughs that grant no capability beyond an already-trusted boundary.
