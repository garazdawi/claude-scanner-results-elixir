# /Users/lukas/code/claude-scanner/elixir/bin/elixir
<!-- sha256: 057efb7c8c5fd2028ed2924cf98c234b323010d6a21080fcd9c452ec5eb6bc01 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/bin/elixir
<!-- sha256: 057efb7c8c5fd2028ed2924cf98c234b323010d6a21080fcd9c452ec5eb6bc01 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-27_

# Security Review — `elixir/bin/elixir`

- **Summary:** The `--pipe-to`/`run_erl` argument escaper (line 234) fails to neutralize newlines, permitting shell command injection into the detached daemon's command line, and the accompanying switch-injection guard is dead code that never fires. Re-review read the three in-folder callers (`iex`, `elixirc`, `mix`) and confirmed they are thin same-trust pass-throughs that neither sanitize argv nor set/preserve `ELIXIR_ERL_OPTIONS` — so the first pass's "outside this file" hedges are resolved against actual neighbour code rather than assumed.

## Security

#### `CSCAN-elixir-pipe-to-newline-injection`
- **Finding ID:** `CSCAN-elixir-pipe-to-newline-injection`
- **Severity:** High
- **Confidence:** high
- **Location:** `elixir:234` (the `for PART in "$@"` escaper) reached from the `--pipe-to` branch (`elixir:180`, `elixir:231`–`elixir:240`).
- **Class:** Command injection.
- **Impact:** In `--pipe-to` (detached daemon) mode, the whole assembled command is passed as a single string to `run_erl`, which executes it through `/bin/sh`. The escaper backslash-escapes every byte outside `[a-zA-Z0-9_/-]` — but a newline is a line separator to `sed`, so it is never matched and passes through **raw**. A raw newline is a command separator in the shell `run_erl` runs, so any argument value carrying a newline splits the command line and starts a second command. The injected token is limited to whitelist-safe characters (`[a-zA-Z0-9_/-]`), which still permits an absolute path (`/tmp/evil`) or a bare command name — i.e. arbitrary code execution as the user running the daemon. This bites when a newline-bearing value comes from a source less trusted than the invoker: untrusted program data passed after the script / `-extra`, or a `--cookie` / `--erl-config` / `--boot` / `--vm-args` / `--name` / `--boot-var` value derived from a file or environment. All of these flow into `"$@"` (lines 211–216, 229) and through the escaper. The escaper exists precisely to make such values safe, and it fails open for a common metacharacter.
- **Reachability via in-folder callers (re-review):** `iex:38` (`exec "$SCRIPT_PATH"/elixir … +iex "$@"`) and `elixirc:43` (`exec "$SCRIPT_PATH"/elixir +elixirc "$@"`) forward the user's argv verbatim, so `iex --pipe-to …` / `elixirc … --pipe-to …` reach this same escaper. Neither wrapper adds nor strips `--pipe-to`, so all three sh entry points inherit the defect.
- **Evidence:** `verified`. Re-confirmed this pass — the real script emits an unescaped newline inside the single `run_erl` command argument:
  ```
  $ DATA="$(printf 'legit\n/tmp/evil_payload')"
  $ ELIXIR_CLI_DRY_RUN=1 sh ./elixir --pipe-to /tmp/p /tmp/l -e "$DATA" | od -c
  ... - e   l e g i t \n / t m p / e v i l _ p a y l o a d \n
  ```
  The `\n` appears raw (no preceding `\`), splitting the `-extra` value across two shell lines. Shell under test confirmed `/bin/sh → /usr/bin/dash`.
- **Suggested fix:** Escape the newline class too, or — to remove the whole class — wrap each part in single quotes instead of backslash-escaping, since everything inside single quotes is literal including newlines: `ESCAPED="$ESCAPED '$(printf '%s' "$PART" | sed "s/'/'\\\\''/g")'"`. Alternatively reject any argument containing a newline before building the `run_erl` command.

#### `CSCAN-elixir-dead-pipe-to-switch-guard`
- **Finding ID:** `CSCAN-elixir-dead-pipe-to-switch-guard`
- **Severity:** Low
- **Confidence:** high
- **Location:** `elixir:184`–`elixir:188` (the `--pipe-to` guard) using `starts_with` at `elixir:94`–`elixir:99`.
- **Class:** Argument injection (non-functional guard).
- **Impact:** The guard is meant to reject a PIPEDIR/LOGDIR that begins with `-` (switch injection). But `starts_with` signals its result via **exit status** (`true`/`false` commands print nothing), while the caller captures **stdout**: `[ "$(starts_with "$RUN_ERL_PIPE" "-")" ]`. `$(...)` is always the empty string, and `[ "" ]` is always false in dash, so the guard never fires — a `-`-leading PIPEDIR/LOGDIR is always accepted. Practical damage is contained because both values are later quoted and used positionally (`mkdir -p "$RUN_ERL_PIPE"`, `run_erl … "$RUN_ERL_PIPE/"`): the worst observed effect is `mkdir` option-injection (e.g. `--pipe-to -m /tmp/l` makes `mkdir -p -m …` error out under `set -e`) and handing an option-shaped positional to `run_erl`; no code execution was found. It is nonetheless a security check that does nothing.
- **Evidence:** `verified`. Re-confirmed this pass:
  ```
  PIPEDIR=-x     captured=[]  => GUARD SILENT (accepts)
  PIPEDIR=-rf    captured=[]  => GUARD SILENT (accepts)
  PIPEDIR=--pipe captured=[]  => GUARD SILENT (accepts)
  [ "" ] is FALSE (guard can never fire)
  ```
- **Suggested fix:** Branch on the exit status directly — `if starts_with "$RUN_ERL_PIPE" "-"; then …` — and/or prefix the values with `./` when handing them to `mkdir`/`run_erl` so a leading `-` can never be read as an option.

### Ruled out

- **Unquoted `$ELIXIR_ERL_OPTIONS` and `$ERL` on line 229 (word-splitting + globbing).** Verified that globbing occurs (`ELIXIR_ERL_OPTIONS='-pa *'` expanded to the directory listing). Ruled out as a finding: word-splitting is the *documented* purpose of `ELIXIR_ERL_OPTIONS` (line 35) and `--erl`, and both sit at the invoker's own trust level. This folder's threat model classes these as trusted-invocation launchers where controlling the environment already means controlling what BEAM boots; no *lower*-trust boundary is crossed in a documented invocation. Re-review checked the in-folder callers for an env/argv trust-lowering step and found none (see "env trust" entry below).

- **`eval` in `erl_set` (line 89) and array reconstruction (line 213).** Verified safe: the idiom `eval "E${E}=\$1"` performs indirect assignment without re-parsing the value — a hostile value `$(PWNED=yes)` was stored literally and did not execute.

- **erl_set values beginning with `-` (`--cookie`/`--name`/`--erl-config`/`--boot`/`--vm-args`/`--boot-var`).** Verified within this file: each value is stored in an `E<n>` slot (line 89), reconstructed into a dedicated positional argv slot (lines 211–216), and delivered to the runtime via `exec "$@"` (line 245) — **no intervening shell**, so a value beginning with `-` cannot break out of shell quoting here. The only path where these values re-enter a shell is `--pipe-to` mode (the `run_erl` escaper), and that path is already covered by `CSCAN-elixir-pipe-to-newline-injection`. Whether `erl` itself mis-parses a `-`-leading *value* as a *switch* is runtime behaviour with no `erl` binary in this folder to exercise; that is a property of the runtime's own argv parser, not a shell-injection sink in this file. Confidence: this file introduces no additional shell re-entry beyond the reported pipe-to path.

- **`SELF`/`SCRIPT_PATH` → `"$SCRIPT_PATH"/../lib/elixir/ebin` (lines 218–229).** The path is quoted, so no splitting. Loading code from the install directory requires controlling the script's location or a symlink in its chain, which is already-privileged; not a lower-trust boundary in documented invocation. (The three in-folder callers likewise derive `SCRIPT_PATH` from `readlink_f "$0"` and none re-validate `../lib/elixir/ebin`, consistent with the standing cross-file invariant — nothing in this folder checks that path.)

- **Missing `set -u` / `set -o pipefail`.** No security-relevant path: there is no verification pipeline whose failure would be masked as success, and the unset variables that matter (`ELIXIR_ERL_OPTIONS`, `RUN_ERL_PIPE`, `OSTYPE`) expand to empty harmlessly under the existing quoted tests.

## Risks / caveats

- **Shell under test:** re-confirmed `/bin/sh → /usr/bin/dash` in this environment; the single-argument `test` semantics that make the guard dead, and the `sed` newline behavior, hold for dash and are POSIX-general.

- **`run_erl` internals were not executed** (`erl`/`run_erl` are not installed here). The injection was demonstrated against the exact string the real script produces. If a given `run_erl` build does *not* route the command through a shell, Finding 1's impact is reduced; the escaper's newline hole remains a real defect regardless.

- **Reachability of Finding 1** is gated on `--pipe-to` mode plus a newline in an argument value sourced from something the invoker treats as data. Re-review confirmed `iex` and `elixirc` forward `"$@"` unfiltered, so `--pipe-to` is reachable through those front-ends too; callers that never use `--pipe-to`, or that guarantee no newline reaches these values, are not affected.

- **Env trust (re-review; first-pass hedge resolved).** The first pass raised a "residual concern" that a higher-privileged wrapper which sanitizes argv but *preserves* `ELIXIR_ERL_OPTIONS` would turn line 229's word-splitting/globbing into a genuine lower-trust-to-sink path — and flagged this as "outside this file." I read the three in-folder callers to check whether any realizes that shape:
  - `iex:36-38` — `readlink_f`, then `exec "$SCRIPT_PATH"/elixir --no-halt --erl "-user elixir" +iex "$@"`. Adds only fixed literal switches; the `--erl "-user elixir"` value is a constant, not attacker-derived. No env set, no argv sanitization.
  - `elixirc:41-43` — `exec "$SCRIPT_PATH"/elixir +elixirc "$@"`. No env set, no argv sanitization.
  - `mix:1-7` — `#!/usr/bin/env elixir` running `Mix.CLI.main()`; no shell-level env or argv handling at all.

  None of the in-folder callers lowers argv trust while preserving `ELIXIR_ERL_OPTIONS`, and none is setuid or otherwise privilege-elevating. So the concern is **not realized by any neighbour in this folder**; it remains only a hypothetical for an *external* setuid/sudo wrapper that does not exist here. Downgraded from a hedged assumption to a verified-negative caveat.

## Did I verify this?

- **Finding 1 (newline injection):** Verified by execution this pass — `ELIXIR_CLI_DRY_RUN=1 sh ./elixir --pipe-to … -e "$(printf 'legit\n/tmp/evil_payload')"` emits a raw `\n` inside the single `run_erl` argument (confirmed via `od -c`). High confidence.
- **Finding 2 (dead guard):** Verified by execution this pass — `starts_with` captured via `$(...)` yields `[]` for `-x`/`-rf`/`--pipe`, and `[ "" ]` is false in dash. High confidence.
- **First-pass "outside this file" hedges:** Resolved by reading the actual neighbour files (`iex`, `elixirc`, `mix`) rather than assuming. Result: (a) erl_set `-`-leading values re-enter a shell only via the already-reported pipe-to path — verifiable within this file; (b) no in-folder caller reintroduces a lower-trust env path. No hedged assumption was upgraded to a finding because the neighbour evidence supports the negative in both cases.

## Dropped in re-review

None. Both findings from the first pass are preserved and re-confirmed by execution.
