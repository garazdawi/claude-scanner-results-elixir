# /Users/lukas/code/claude-scanner/elixir/bin/elixir
<!-- sha256: 057efb7c8c5fd2028ed2924cf98c234b323010d6a21080fcd9c452ec5eb6bc01 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T07:36:51Z_

## Summary

The `--pipe-to` switch-rejection guard is dead code because it tests the (always-empty) stdout of `starts_with` instead of its exit status, so a switch supplied where a PIPEDIR/LOGDIR is expected is never rejected.

## Bugfixes

#### `CSCAN-elixir-starts-with-guard-dead`
- **Finding ID:** `CSCAN-elixir-starts-with-guard-dead`
- **Severity:** Low
- **Confidence:** high
- **Location:** `elixir/bin/elixir:184-188` (`--pipe-to` case arm), helper `starts_with` at `elixir/bin/elixir:94-99`
- **Class:** Dead code and unreachable branches
- **Impact:** A user (or a wrapping script/`mix release` launcher) who invokes `--pipe-to` with a missing directory argument — so that the next token, a switch like `-foo`, lands in `PIPEDIR`/`LOGDIR` — is meant to get the clear diagnostic `--pipe-to : PIPEDIR cannot be a switch` and a clean `exit 1`. Because the guard can never be true, the script instead falls through to `mkdir -p "-foo"`, which the user sees as the confusing, unrelated error `mkdir: invalid option -- 'f'` (and under `set -e` the script dies there). The intended, friendly error path is unreachable.
- **Evidence:** `verified`.
  - `starts_with` emits nothing on stdout — its `case` arms run the builtins `true`/`false`, which set exit status only. So `$(starts_with "$RUN_ERL_PIPE" "-")` is always the empty string and `[ "" ]` is always false:
    ```
    $ RUN_ERL_PIPE="-oops"
    $ if [ "$(starts_with "$RUN_ERL_PIPE" "-")" ]; then echo FIRED; else echo "NOT FIRED: [$(starts_with "$RUN_ERL_PIPE" "-")]"; fi
    NOT FIRED: []
    $ if starts_with "$RUN_ERL_PIPE" "-"; then echo "correct idiom would reject"; fi
    correct idiom would reject
    ```
  - The downstream error the user actually hits instead of the intended message:
    ```
    $ mkdir -p "-x"
    mkdir: invalid option -- 'x'
    exit=1
    ```
- **Suggested fix:** Test the helper's exit status, not its output: replace `if [ "$(starts_with "$RUN_ERL_PIPE" "-")" ]; then` with `if starts_with "$RUN_ERL_PIPE" "-"; then` and likewise `elif starts_with "$RUN_ERL_LOG" "-"; then`.

## Risks / caveats

- I ran the offending fragment directly but did not execute the full launcher end-to-end against a real `run_erl`; the guard-is-dead behavior is verified in isolation, and the surrounding `--pipe-to` flow is read-only reasoning.
- `ELIXIR_ERL_OPTIONS` and `$ERL` are deliberately left unquoted at line 229 so the shell re-splits accumulated Erlang switches into separate argv words; this is intended array-reconstruction, not a quoting defect, and I did not flag it.

### Ruled out
- **Unquoted install path with spaces (line 229).** `-elixir_root "$SCRIPT_PATH"/../lib` and `-pa "$SCRIPT_PATH"/../lib/elixir/ebin` keep `$SCRIPT_PATH` inside quotes, and the trailing `/../lib…` literals contain no spaces, so the whole path survives as one word. Verified by substituting a spaced path: the expansion stays a single argument.
- **User values with spaces (`.exs` path, `-e`, `--cookie`, `--name`).** Program args go through the `set -- "$@" "$1"; shift` rotate, which preserves each argument whole; distribution/config values go through `erl_set`/`E$n` slot storage, also whole. Confirmed by tracing the copy loop and the `erl_set` eval — each user token is stored and re-emitted as one field.
- **`readlink_f` symlink-loop recursion (lines 69-77).** No cycle guard, but reaching it requires the installed `elixir` executable itself to be a symlink loop — an install pathology outside the documented invocation surface; not a caller-supplied input path.
- **`$OSTYPE` is a bashism under `#!/bin/sh` (line 221).** Under a POSIX `sh` where `OSTYPE` is unset, `[ "$OSTYPE" = "cygwin" ]` is simply false (no `set -u` here), which is the correct non-cygwin default; it degrades safely rather than failing.
- **Copy/shift (`C`/`S`) arities (lines 112-209).** Cross-checked every `case` arm against each flag's documented token count (`-e`/`-r`→2, `--rpc-eval`→3, `--boot-var`/`--pipe-to`→3, single-token flags→1, value flags→2); all match the USAGE contract, so no argument-count desync found.

