<!-- rfc-governed: no -->
<!-- standards: -->

# Documented-Contract / Behavioural-Obligation List — `elixir/bin`

## Nature of the folder

This folder contains the **Elixir language launcher scripts**, not `.erl` source modules. It holds POSIX `/bin/sh` scripts (`elixir`, `elixirc`, `iex`, `mix`) and their Windows counterparts (`*.bat`, `mix.ps1`). Their job is to parse command-line switches, marshal them into Erlang VM (`erl` / `run_erl` / `to_erl`) invocations, resolve the script's own path (`readlink_f`), and `exec` the runtime. There is no network protocol, no wire/message format, no state machine, and no cryptographic algorithm implemented here — only argument shuffling and process launch.

## Applicable standards / contracts

### External standards
**None.** No IETF RFC, ITU-T / W3C / ISO-IEC recommendation, or NIST/FIPS publication governs this folder. The `--cookie` / `--sname` / `--name` distribution options and `--boot`, `--vm-args`, `--pipe-to` options are pass-throughs to Erlang/OTP tooling (`erl`, `run_erl`, `to_erl`); the Erlang distribution protocol itself is implemented elsewhere in the runtime, not in these launcher scripts. There are consequently **no external protocol/algorithm conformance obligations** for the compliance pass to adjudicate — this is the empty-list case.

**Standards references gathered from the tree:** the scan reported *(no standards references found in the application tree)*, and reading the folder confirms it — there is nothing to account for or explain away.

### OTP-internal / shell contracts (per-file context, NOT compliance obligations)
The only "contracts" here are the launcher's own documented CLI behaviour and POSIX-shell portability conventions. These are guidance for the per-file reviewer, not standard-conformance obligations:

- The `USAGE` heredoc in each script is the documented switch surface; the parser's `case` arms (copy-count `C` vs shift-count `S`) must stay consistent with it.
- Scripts are `#!/bin/sh` and use `set -e`; they must remain POSIX-portable (no bashisms) and must not word-split unquoted user-supplied paths.
- The `ERTS_BIN=$ERTS_BIN` line carries an in-file MUST-NOT-EDIT comment because Elixir releases rewrite it — a mechanical (release-tooling) contract, not an external standard.

## Checkable sub-claims

1. **ARG_QUOTING_PRESERVED** — User-supplied values (file paths, `-e` commands, cookie, node name) must stay individually quoted through the array-mimicking `erl_set`/`E$n` marshalling and the final `set -- … "$@"`, so paths with spaces reach `exec` intact. (Shell-safety convention; `elixir:88-91,211-216,229`.)
2. **PIPE_TO_SWITCH_REJECTED** — `--pipe-to` must reject a PIPEDIR/LOGDIR that begins with `-` (a switch masquerading as a directory) before `mkdir`/`run_erl`. (`elixir:184-188`.)
3. **SHIFT_COUNT_MATCHES_ARITY** — Each option's copy (`C`) / shift (`S`) count must match the number of tokens it actually consumes; an off-by-one drops or duplicates a following argument. (`elixir:112-209`.)
4. **SYMLINK_RESOLVED_SAFELY** — `readlink_f` must resolve chained symlinks to locate the real script/`../lib` root; failure would load the wrong `elixir/ebin`. (`elixir:69-77,218-229`.)
5. **ERTS_BIN_LINE_UNMODIFIED** — The `ERTS_BIN=$ERTS_BIN` line must be left exactly as-is per its in-file comment (release tooling rewrites it). (`elixir:224-227`.)
6. **SET_E_ERROR_PROPAGATION** — With `set -e`, a failed intermediate command must not be silently swallowed; error paths (`echo … >&2 && exit 1`) must terminate with non-zero status. (`elixir:7,185-187`.)
7. **STANDALONE_OPTS_EXCLUSIVE** — Standalone switches (`--help`, `--version`, `--short-version`) must be honoured only when given alone, matching the documented "can't be combined" rule. (`elixir:11,79-82`; USAGE lines 64.)
8. **NO_UNQUOTED_GLOB_EVAL** — `eval`-based slot storage (`eval "E${E}=\$1"`) must not expand user data as code or globs; the escaping in the `--pipe-to` `sed` filter must cover shell metacharacters. (`elixir:88-90,231-235`.)
9. **WINDOWS_PARITY** — The `.bat` / `.ps1` variants should parse the same documented switch set as the `sh` scripts; a switch handled on one platform but silently dropped on the other is a defect. (`elixir.bat`, `mix.ps1` vs `elixir`, `mix`.)

## Per-file checking instructions

There are **no external protocol/algorithm obligations** for this folder — the RFC-compliance pass has nothing to adjudicate here (empty `standards` marker). For the per-file review, treat the numbered sub-claims as **shell-correctness and CLI-contract** checks, not standards-conformance checks: for each claim that plausibly applies to the file under review, look for the code that satisfies it (correct shift/copy arithmetic, quoting of user values, switch-rejection guards, symlink resolution). If a launcher script parses a documented switch with the wrong arity, fails to quote a user-supplied path before `exec`, or drops a guard present in its sibling platform script, report it as a finding under the appropriate shell-safety / argument-injection failure class — not as an RFC gap.