# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell/process.ex
<!-- sha256: f1660332f79a6fb08a64bcd6f4b8868196b259ce8825e04ba93bd3b0071bc2ba -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell/process.ex
<!-- sha256: f1660332f79a6fb08a64bcd6f4b8868196b259ce8825e04ba93bd3b0071bc2ba -->

_Re-analysed on 2026-08-27_

All public functions traced and driven. This is a re-review resolving the first-pass hedges. Findings summary unchanged:

- `info/1`,`error/1` → `format/1` runs `IO.ANSI.format(msg, false)` on **caller-supplied** chardata; raises `ArgumentError` on a non-ANSI atom / bad iodata (executed, below), but that is a caller-contract violation, and `Mix.Shell.Process` is only the active shell when a caller opts in.
- `prompt/1`,`yes?/2` return mailbox input verbatim with no validation — seeding `{:mix_shell_input, …}` requires a local process holding the pid (the documented test contract), not a dependency/network/archive boundary.
- `cmd/2` forwards the caller's `command` into `Mix.Shell.cmd` → `System.shell` (the injection sink lives in `mix/shell.ex:164`, now read and confirmed — not here).
- `message_target/0` reads `$callers` from the process dictionary — local runtime state.

None of these route dependency/archive/git/network/file-derived data into a dangerous sink *within this file*.

## Summary
`Mix.Shell.Process` is a small opt-in test shell that forwards inert message tuples to a process mailbox; no dependency-, archive-, network-, or file-derived data reaches a dangerous sink inside this file, and I found no security vulnerabilities.

## Security

No findings

### Ruled out

- **`cmd/2` command injection** — `cmd/2` (`process.ex:168`) forwards the caller's `command` to `Mix.Shell.cmd/3`. I read the neighbour `lib/mix/shell.ex:145–168`: the binary-command branch runs `System.shell(command, options)` (`shell.ex:164`), and the `{command, args}` branch runs `System.cmd/3` (`shell.ex:163`). The `System.shell` sink is in `mix/shell.ex`, not this file. This file's own contribution is only the streaming callback `fn data -> send(message_target(), {:mix_shell, :run, [data]})`, which delivers an inert message tuple — no interpreter. `command` originates from the calling Mix task, not from dependency/archive/network data traversing this file. Confidence: High.
  - Axes examined: which branch of `Mix.Shell.cmd` executes (both — `System.shell` for binary, `System.cmd` for tuple); what this file adds downstream of the sink (only an inert `send`). Not examined: none.

- **`prompt/1` / `yes?/2` return unvalidated input** — executed: with `{:mix_shell_input, :prompt, {:not, "a string"}}` seeded, `prompt/1` returns `{:not, "a string"}` verbatim; with `{:mix_shell_input, :yes?, :maybe}` seeded, `yes?/2` returns `:maybe` verbatim (neither is coerced to the documented `binary`/`boolean`). Seeding that message requires a local process already holding the target pid; this is the documented testing contract, not a remote/dependency trust boundary. Cross-node message injection needs a shared cookie = full trust, out of scope. Confidence: High.
  - Axes examined: non-string `prompt` response; non-boolean `yes?` response (both returned verbatim). Not examined: the "who can seed the mailbox" axis — cannot be tried, because seeding requires code co-resident in the same BEAM process, which is already operator-trusted; this is a scope exclusion, not a narrowed guard.

- **`format/1` `ArgumentError`** — executed on the source-built toolchain (OTP 29 / Elixir): `["x", :not_ansi]`, `[-1]`, and `[0x110000]` each raise `ArgumentError` in `IO.ANSI.format(false)` before `IO.iodata_to_binary/1`; valid nested chardata (`["a", [?b, "c"], :red]`) yields `"abc"`. The message is the caller's own argument (violating the documented `IO.ANSI.ansidata()` contract). Dependency-derived data that reaches `info/1`/`error/1` is a **binary string** (valid chardata → no crash); only a caller passing a bad atom/negative integer triggers the crash, and that is a caller-contract violation, not attacker-controlled. This shell is not on the `mix compile`/`deps.get` default path. Excluded by scope. Confidence: High.
  - Axes examined: bad atom, negative integer, out-of-range codepoint, valid nested chardata (all run). Not examined: none material — the sink only raises `ArgumentError`, and the reachable attacker input class (binaries) is proven non-crashing.

- **`message_target/0` `$callers` routing** — read `process.ex:177–182`: returns the head of the `$callers` process-dictionary list, or falls back to `self()` for empty/non-list values. `$callers` is set by the local OTP/`Task` runtime; poisoning it requires code already executing in the same process (operator-trusted). No trust boundary crossed; standard Elixir caller-tracking idiom. Confidence: High.
  - Axes examined: list-head vs. empty/non-list fallback (read from source). Not examined: the "attacker sets `$callers`" axis — cannot be tried, because writing the process dictionary requires in-process code, which is already trusted; scope exclusion, not a narrowed guard.

- **`print_app/0` app-name interpolation** — `"==> #{name}"` (`process.ex:79`) interpolates `Mix.Shell.printable_app_name()` (→ `Mix.ProjectStack.printable_app_name()`, `shell.ex:114–117`) into a string that becomes a message-tuple element. It reaches no downstream interpreter (shell/query/markup); it is inert data delivered to the target process. No injection sink. Confidence: High.

- **`flush/1` callback + selective `receive`** — the callback is caller-supplied and the drained `{:mix_shell,…}`/`{:mix_shell_input,…}` messages are inert; recursion is bounded by current mailbox size (`after 0` terminates when no matching message remains). No attacker-influenced sink. Confidence: High.

## Risks / caveats
- The OS-command-execution behavior reachable *through* this file's API (`cmd/2`) is implemented in `lib/mix/shell.ex` (`System.shell`, `shell.ex:164`); whether a Mix task feeds it attacker-influenced `command` strings belongs to a review of that file and of the tasks calling `Mix.shell().cmd/2`, not this one.
- **Resolved (first-pass hedge on install path):** I grepped the entire `lib/mix/lib` tree for `Mix.Shell.Process`; the only occurrences are this module's definition and its own docstring example (`process.ex:5,27`) plus two documentation mentions in `mix.ex:551,572`. No code path in Mix installs this shell by default — `mix.ex:553` documents the default as `Mix.Shell.IO`. Within Mix's own source, `Mix.Shell.Process` is strictly opt-in via an explicit `Mix.shell(Mix.Shell.Process)` call (test harnesses). This narrows, but does not touch, third-party callers outside this tree.

## Did I verify this?
- `format/1` crash axes (bad atom, negative int, out-of-range codepoint, valid nested chardata): **executed** on OTP 29 / Elixir — all four behaviours as reported.
- `prompt/1` / `yes?/2` verbatim return of non-string / non-boolean input: **executed** — returned `{:not, "a string"}` and `:maybe` unchanged.
- `cmd/2` → `Mix.Shell.cmd` → `System.shell`/`System.cmd` sink location: **verified by reading** `lib/mix/shell.ex:145–168`.
- `printable_app_name` origin: **verified by reading** `lib/mix/shell.ex:114–117`.
- Opt-in-only install path within Mix: **verified by grep** over `lib/mix/lib` (no non-doc installer).
- The two remaining `Not examined` axes (who can seed the mailbox; who can set `$callers`) are **untryable** — both require in-process, operator-trusted code, so they are scope exclusions and correctly remain out of scope rather than narrowed.

### Dropped in re-review
None. All first-pass ruled-out entries are retained; every prior hedge was either resolved by reading the named neighbour (`mix/shell.ex`) / grepping the tree, or by executing the sink axis. No assumption required upgrading to a finding.
