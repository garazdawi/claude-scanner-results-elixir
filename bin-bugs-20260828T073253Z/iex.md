# /Users/lukas/code/claude-scanner/elixir/bin/iex
<!-- sha256: 5f5756abaf69e394de4e4e6d1c9f80bf4258c55509af4403a199abf5e7afcfdc -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/bin/iex
<!-- sha256: 5f5756abaf69e394de4e4e6d1c9f80bf4258c55509af4403a199abf5e7afcfdc -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

The `iex` launcher is a delegation stub: it prints IEx-specific usage, resolves its own path via `readlink_f`, and `exec`s the sibling `elixir` script with `--no-halt --erl "-user elixir" +iex` prepended before `"$@"`. Every user- and path-derived expansion is individually quoted (`"$@"`, `"$SELF"`, `"$SCRIPT_PATH"`), so no defect reachable through the documented `iex [options] [.exs file] [data]` interface survives review. In this pass I read the delegated `elixir` script and confirmed that all three prepended tokens are recognised there with correct arity and ordering, so the first pass's "out of scope" hedge is now retired as verified.

## Bugfixes

No findings

## Risks / caveats

- **No symlink-cycle guard in `readlink_f` (lines 26–34).** If the installed `iex` executable is itself part of a symlink loop, `readlink_f` recurses until the shell's recursion limit aborts it. Reproduced with an isolated copy of the function against a two-node symlink cycle:

  ```
  $ timeout 5 /tmp/rf.sh /tmp/loopa; echo "exit=$?"
  /tmp/rf.sh: 7: Maximum function recursion depth (1000) reached
  exit=2
  ```

  Confidence: this is a real behaviour but **not a reportable finding**. It is reachable only when the launcher's *own install path* is a self-referential symlink (a corrupted installation), never through any command-line argument, environment value, or input file crossing the documented public boundary. The identical guardless resolver is shared verbatim by `elixir`, `elixirc`, and `mix` by design (confirmed: `elixir` lines 69–77 are byte-for-byte the same function). Not untrusted data crossing a public boundary → risk, not finding.

## Ruled out

- **Delegation forwards an unrecognised or wrong-arity flag to `elixir`** — *Upgraded from hedged assumption in first pass; now verified by reading the neighbour.* The first pass declared switch correctness "out of scope for this file and must be judged in `elixir`." I read `/Users/lukas/code/claude-scanner/elixir/bin/elixir` and confirmed each token `iex` prepends (line 38) is handled by that parser:
  - `--no-halt` → matched at `elixir:120-122`, `C=1` (copies the flag itself, consumes no following token). Correct arity.
  - `--erl "-user elixir"` → matched at `elixir:145-148`, `S=2`, `ERL="$ERL $2"`. The single argv element `-user elixir` is stored and intentionally word-split by the later unquoted `$ERL` expansion (`elixir:229`) into `-user elixir`. Correct.
  - `+iex` → matched at `elixir:116-119`, `C=1`, and this is the **only** arm that sets `MODE="iex"`, which at `elixir:222` suppresses the `-s elixir start_cli` prepend. Correct.
  - **Ordering invariant holds:** `iex` places `+iex` ahead of `"$@"`, and every token before it (`--no-halt`, `--erl …`) is a recognised flag, so the parser never reaches the `*)` catch-all (`elixir:190-197`) before consuming `+iex`. `MODE=iex` is therefore always set regardless of the user's own arguments. Confirmed by inspection of the `while` loop's copy/shift bookkeeping.
  - Confidence: high. No divergence found. The IEx-only switches (`--dbg`, `--dot-iex`, `--remsh`) are genuinely handled in `elixir` (`elixir:123`, `C=2`), so nothing `iex` documents is dropped downstream.
- **Unquoted expansion / word-splitting on a path with spaces** — verified clean. Invoked via a symlink whose path contains a space; `SELF`, `SCRIPT_PATH`, and the resolved script path all survived intact (`SELF=[/tmp/iex test/bin/iex]`, `SCRIPT_PATH=[/tmp/iex test/bin]`). Line 38 forwards arguments as `"$@"`, preserving argument boundaries including embedded spaces. Confidence: high (executed).
- **`cd` failure swallowed** — `set -e` (line 7) is active and the `cd` in `readlink_f` (line 27) is not in a suppressed context (its stdout, not exit status, is redirected to `/dev/null`), so a failed `cd` aborts with non-zero status rather than resolving against the wrong directory. Confidence: high.
- **Bashisms under `#!/bin/sh`** — the script uses only POSIX constructs (`[ ]`, `basename`/`dirname`, `readlink` without `-f`, `pwd -P`); no `[[`, arrays, `local`, or `echo -e`. `readlink` is deliberately called without the non-portable `-f` flag because resolution is done manually. Confidence: high.
- **Empty `$1` on the help/`-h` test (line 9)** — with no arguments, `[ "$1" = "--help" ]` evaluates `[ "" = "--help" ]`, which is well-formed and false; the quoting makes it safe. Confidence: high.

## Did I verify this?

- **Delegation correctness:** Yes — read `elixir` (the neighbour named in the standing context) and cross-checked the arity (`C`/`S`) and ordering of every token `iex` prepends against that parser's `case` arms. This retires the first pass's "out of scope" hedge with executed/inspected evidence rather than assumption.
- **Symlink cycle:** Yes — executed an isolated copy of `readlink_f` against a real two-node symlink loop and captured the recursion-depth abort. Confirmed the same function is duplicated in `elixir`.
- **Quoting/word-splitting on spaced paths:** Yes — executed via a space-containing symlink path and observed intact expansions.
- **Windows parity (`iex.bat`):** No — the `.bat` counterpart is a separate file; divergence between it and this script is a legitimately different file's finding and was not examined here.

## Dropped in re-review

- **"Delegation, not divergence" caveat (first-pass Risks/caveats).** Not dropped as a concern but **reclassified**: the portion claiming switch correctness is "out of scope for this file" is removed as a hedge and replaced by the verified "Ruled out" entry above, because I read `elixir` and confirmed the forwarded flags are handled with correct arity and ordering. The only genuinely out-of-scope item — `iex.bat` Windows parity — is retained under "Did I verify this?" as a not-examined sibling-file concern.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Empty `$1` on the help test (line 9)** — with no arguments, `[ "$1" = "--help" ]` evaluates `[ "" = "--help" ]`, which is well-formed and false; the quoting makes it safe.
