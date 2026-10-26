# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/port.ex
<!-- sha256: bd87bcf528512ca6333dfe9bec299e77a04f98a3621ea64a6b403e1374c4e54d -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/port.ex
<!-- sha256: bd87bcf528512ca6333dfe9bec299e77a04f98a3621ea64a6b403e1374c4e54d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (hedge-resolution pass over the first-pass report)._

Two hedges from the first pass are resolved here with executed evidence rather than assumption:

1. *"I did not exhaustively enumerate ERTS's exact 'needs-a-shell' character set"* — resolved. The premise was wrong: `open_port({spawn, Cmd})` on Unix does **not** conditionally invoke a shell on certain characters. It runs `/bin/sh -c "exec Cmd"` **unconditionally**. I proved this by pushing a metacharacter-free single-word command through the real ERTS (`open_port({spawn, "definitely_no_such_program_12345"}, ...)`) → `sh: 1: exec: definitely_no_such_program_12345: not found`. The literal string `sh: 1: exec:` from the shell itself shows both that a shell always runs and that the command is prefixed with `exec `. So there is no character-set boundary to enumerate; the shell is always present.
2. The first-pass "Ruled out" claim that `;` and newline *"are consumed by Erlang's own tokenizer as terminators and never reach the shell"* is **incorrect** and is corrected below. They reach `/bin/sh` like everything else; the reason a chained second command usually does not execute is the `exec ` prefix (which replaces the shell process before the second command runs), not any Erlang-side tokenization.

## 1. Summary
`Port` is a faithful thin wrapper over Erlang's `:erlang.*` port primitives. The one security-relevant defect is unchanged: the `Port.open/2` documentation mis-frames `{:spawn, cmd}` as a direct `$PATH` program lookup and scopes its only injection warning to Windows, while in reality **every** `{:spawn, cmd}` on Unix is executed via `/bin/sh -c "exec <cmd>"`, making the command string a shell-injection sink. The executed evidence in this pass strengthens the finding (the shell is unconditional, not metacharacter-gated) and keeps its confidence high.

## 2. Security

#### `CSCAN-port-spawn-shell-injection`
- **Finding ID:** `CSCAN-port-spawn-shell-injection`
- **Severity:** Low
- **Confidence:** high
- **Location:** `port.ex:231` (`open/2`); moduledoc `### spawn` section, lines 107–121, together with the Windows-only warning block at lines 86–105.
- **Class:** Command execution (path through a shell)
- **Issue:** `open/2` forwards `{:spawn, command}` verbatim to `:erlang.open_port/2`. On Unix, ERTS executes this as `/bin/sh -c "exec <command>"` **for every spawn command, unconditionally** — verified this pass (see Evidence). The moduledoc instead describes `:spawn` as a direct program lookup — *"`:spawn` will retrieve the program name from the argument and traverse your operating system `$PATH` … looking for a matching program"* (lines 116–117) — and its only injection warning (lines 86–105) is explicitly scoped to **Windows** `.bat`/`.com` files. Nothing tells the reader that on Unix a `:spawn` command containing `$()`, backticks, `` ` ``, `|`, `>`, `*`, `~`, or `$VAR` is expanded and executed by a shell, so untrusted content interpolated *anywhere* in the command string is shell command injection (CWE-78), not merely argument injection.
- **Impact / threat:** A caller who builds a `:spawn` command by interpolating data from the network, a file, or a dependency — reading these docs as "direct `$PATH` lookup, only Windows is risky" — gets arbitrary command execution on Unix. Trust boundary crossed: external data → `Port.open({:spawn, …})` → `/bin/sh`. The behavior lives in ERTS and is intended; the defect *in this file* is that its own public-API security guidance is incomplete and actively misleading.
- **Evidence:** `verified` (this pass, executed against the real ERTS on this sandbox; OTP 29, `/bin/sh` = dash):
  - **Shell invocation is unconditional and `exec`-prefixed** — `open_port({spawn, "definitely_no_such_program_12345"}, …)` (no metacharacters) → `sh: 1: exec: definitely_no_such_program_12345: not found`, exit 127. The `sh: 1: exec:` text is emitted by the shell, proving `/bin/sh -c "exec <cmd>"` runs even for a metacharacter-free command. `echo $0` → `sh`.
  - **Arbitrary execution via expansion (unaffected by the `exec` prefix):** `echo x$(id -u)` → `x1000`; `` echo x`id -u` `` → `x1000`; `echo $(echo INJ); echo AFTER` → `INJ` (command substitution runs during parsing regardless of exec). `echo $HOME` → `/home/claude`; `echo ~` → `/home/claude`; `echo /tmp/csi_redir*` glob-expanded to `/tmp/csi_redir_test`; `echo hi>/tmp/csi_redir_test` created the file (redirection); `echo a|cat` → `a` (pipe parsed by sh); `true & echo bg` → `bg` (background + trailing command).
  - **`exec` prefix limits chaining (corrects first pass):** `true; echo AFTER` → empty (exec replaces the shell before `; echo AFTER`); `echo start && echo mid` → `start` only; `false || echo alt` → exit 1, empty. So `;`, `&&`, `||`, and newline after the first exec'd word generally do not run a second command — but this is the `exec` prefix, not Erlang tokenization, and it does **not** protect against `$()`/backticks/globs/redirs/pipes/expansion above.
- **Suggested fix:** In the `### spawn` docs, state that on Unix the command is always run via `/bin/sh -c "exec <command>"` and is therefore subject to shell command injection (command substitution, globbing, redirection, variable/tilde expansion all apply), and that untrusted data must never be interpolated into a `:spawn` command on **any** platform (broaden the Windows-only warning). Point callers needing untrusted input at `{:spawn_executable, path}` with an explicit `args:` list (passed as an argv array, never reaching a shell) or at `System.cmd/3`.

### Ruled out
- **`close/1`, `command/3`, `connect/2`, `monitor/1`, `demonitor/2`, `list/0`** — each is a one-line verbatim delegate to `:erlang.*` with no data transformation, no path/atom/eval sink introduced in this file. Basis: read (`port.ex:243–350`). Axes examined: data transformation, sink introduction | Not examined: none (no transformation exists to differ from the delegate).
- **Atom-table exhaustion** — no `String.to_atom`/`binary_to_atom`/`Module.concat` anywhere; `info/2` and `monitor/1` accept caller-supplied atoms but never create them from bytes. Basis: read of the full file. Axes examined: atom creation from bytes | Not examined: none.
- **`info/1,2` + `nilify/1`** — the `:registered_name`/`:undefined`/`[]` handling (`port.ex:290–354`) is correctness normalization (`:undefined`→`nil`, `[]`→`{:registered_name, []}`), not a security predicate; all branches are reachable, so no dead defensive code. Basis: read. Axes examined: reachability, security-relevance | Not examined: none.
- **`;` / newline as an *additional* shell-injection vector** — **corrected from the first pass.** They are *not* filtered by Erlang tokenization and *do* reach `/bin/sh`; the reason a trailing chained command does not run is the `exec ` prefix that ERTS prepends (`/bin/sh -c "exec <cmd>"`), which replaces the shell process before the second command executes. They therefore add no *new* multi-command capability beyond the finding above, but the mechanism claimed in the first pass was wrong. Basis: executed (`true; echo AFTER` → empty; `nosuchcmd_xyz; echo AFTER` → `sh: 1: exec: nosuchcmd_xyz: not found`). Axes examined: reaches-shell, second-command-executes | Not examined: none.
- **Windows `.bat`/`.com` argument injection (BatBadBut-class)** — not a gap: already documented at length (lines 86–105). Noted as the existing mitigation, not a new finding. Axes examined: documentation presence | Not examined: runtime behavior on Windows (no Windows host in this sandbox — untestable here, so this remains an assertion about the *docs* being present, not about the runtime).
- **`command/3` data payload** — `data` is iodata written to the spawned program's stdin; how the program interprets it is the program's concern, and the caller already chose that program. No sink introduced here. Basis: read (`port.ex:255–257`). Axes examined: sink introduction | Not examined: none.

## 3. Did I verify this / risks / caveats
- **Resolved hedge (was "did not exhaustively enumerate the needs-a-shell character set"):** there is no such character set. `{:spawn, cmd}` is executed via `/bin/sh -c "exec <cmd>"` unconditionally on Unix — proven by running a metacharacter-free command through the real ERTS and observing the shell's own `sh: 1: exec:` error. The first-pass framing ("precise trigger boundary is an ERTS implementation detail that may vary") is superseded: the boundary is "always," so no enumeration is needed for the finding to hold.
- **Corrected first-pass error:** the ruled-out claim that `;`/newline are "handled by Erlang's tokenizer as terminators and never reach the shell" is false. They reach the shell; the `exec` prefix is what usually suppresses a trailing chained command. Corrected above with executed evidence.
- **Environment:** all behavior verified on this sandbox (Linux, Erlang/OTP 29, `/bin/sh` = dash) using the toolchain at `/opt/elixir` / `/usr/local`. The `exec`-prefix mechanism is confirmed by the literal `exec` token in dash's error output and by `$0` → `sh`.
- **Not testable here:** the separately-documented Windows argument-splitting behavior (lines 86–105) — no Windows host — was not re-verified this pass and stays an assertion about the docs, not the runtime.
- This module adds no logic to the security-sensitive path, so no code change *in this file* is needed to make the primitive safe — a shell-free alternative (`:spawn_executable` with `args:`, or `System.cmd/3`) already exists. The single reportable defect is documentation completeness for the module's own public API; rated Low because exploitation still requires the caller to forward untrusted data into `open/2`, but the misleading framing meaningfully raises the odds a caller does exactly that.

### Dropped in re-review
- No findings dropped. The one finding (`CSCAN-port-spawn-shell-injection`) is retained and strengthened. One *ruled-out* entry (`;`/newline) had its stated rationale corrected — from "Erlang tokenizer terminators, never reach shell" to "reach the shell, suppressed by the `exec` prefix" — but its conclusion (not an additional injection vector beyond the finding) is unchanged.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`;` / newline as an injection vector** — ruled out as *additional* shell-injection surface: verified they are handled by Erlang's tokenizer as command terminators and do not reach `/bin/sh` (so they add nothing beyond the finding above).
