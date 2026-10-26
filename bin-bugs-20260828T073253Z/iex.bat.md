# /Users/lukas/code/claude-scanner/elixir/bin/iex.bat
<!-- sha256: c4c9c3601401b600043def0b209d6eb0627a1ae8449a78ade3dca310541b9557 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/bin/iex.bat
<!-- sha256: c4c9c3601401b600043def0b209d6eb0627a1ae8449a78ade3dca310541b9557 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T07:41:49Z_

## Summary

`iex.bat` is a thin delegator that forwards its arguments to `elixir.bat` with the IEx-specific flags prepended (`--no-halt --erl "-user elixir" +iex %*`), and I found no defect reachable through its documented invocation. This re-review resolved the first pass's one "out of scope" hedge by reading the delegation target (`elixir.bat`) and the POSIX sibling (`iex`); the delegation contract is now **verified**, not assumed. No new bug was uncovered. The cmd.exe-quoting axes underlying the "Ruled out" entries remain untried because the sandbox has no `cmd.exe` interpreter (POSIX `sh` only), so those dismissals stay `asserted` rather than being promoted to `verified`.

## Bugfixes

No findings.

## Risks / caveats

- **Execution-environment limitation (unchanged).** The sandbox provides POSIX `sh`, not `cmd.exe`, so none of the batch-quoting reasoning below could be confirmed by *running* the file. Every cmd.exe-semantics judgment remains `asserted`, grounded in documented cmd.exe behaviour (`%1` vs `%~1` quoting, `%*` vs `shift`) and cross-checked against the shipped sibling `elixir.bat`, which uses the identical idioms. Reading `elixir.bat` did **not** clear these axes — it tells me what the target parser does, not how cmd.exe tokenises a quoted argument.

- **Delegation contract — verified (was "out of scope" in first pass).** I read `elixir.bat`. The IEx-exclusive switches documented in `iex.bat`'s usage text are in fact parsed by the delegation target: `--remsh` (`elixir.bat:117`), `--dot-iex` (`elixir.bat:118`), and `--dbg` (`elixir.bat:119`), each consuming one following token via `shift`. The three flags `iex.bat` itself prepends are all recognised there: `--no-halt` (`elixir.bat:114`, no arg), `--erl` (`elixir.bat:131`, consumes `-user elixir` via `%~1`+`shift`), and `+iex` (`elixir.bat:102`, sets `useIEx=1`). Because all three prepended tokens are recognised and consumed in order *before* the user's `%*`, the `MODE=iex` ordering invariant holds: `useIEx` is set before the first unrecognised positional falls through to `:run` (`elixir.bat:134`), which is what gates the `-s elixir start_cli` prefix (`elixir.bat:136-137`). This is no longer an assumption — it is confirmed against the source. Confidence: **high** (direct read of both files).

- **`--help` behaviour diverges from the POSIX `iex` script — verified.** I read `iex` (sh). `iex.bat:14-26` prints usage to **stdout** and exits **0** (`goto end` → `endlocal`); `iex:9-24` prints to **stderr** (`cat <<USAGE >&2`) and exits **1** (`exit 1`). The divergence is real and now source-confirmed, but each side follows its own platform's help convention and the help text is complete on both, so this is a cosmetic/convention difference, not a functional failure. Not reported as a bug. Confidence: **high** (both sources read); severity: cosmetic.

### Ruled out

- **`""%1""==""--help""` breaking on a spaced path argument (`iex.bat:8-11`).** Concern: an unprotected space inside `""%1""` splits cmd's `IF` string1 and aborts the line. Rejected on the cmd.exe mechanism: `%1` (unlike `%~1`) preserves the user's surrounding quotes, so `iex "my dir\a.exs"` expands `%1` to `"my dir\a.exs"` and `""%1""` to `"""my dir\a.exs"""`, in which the space sits inside a quoted region and is not word-split; an unquoted argument cannot contain a space. The comparison evaluates false and falls through to `:run`. Same idiom `elixir.bat:9-14` ships with (confirmed by read). Status: `asserted`. Axes examined: none (no cmd.exe interpreter available). Not examined: cmd.exe `IF`-tokenisation of an argument containing an embedded literal quote, an argument containing `%`/`!`, an argument that is a lone `"` — I cannot push these through a real `cmd.exe` in this sandbox, so the entry stays `asserted`.

- **Double backslash in `"%~dp0\elixir.bat"` (`iex.bat:29`).** `%~dp0` already ends in `\`, yielding `...\bin\\elixir.bat`. Rejected: the Win32 path layer collapses repeated separators, so `\\` resolves identically to `\`; the surrounding quotes handle an install path containing spaces (e.g. `C:\Program Files\...`). Benign. Status: `asserted`. Axes examined: none. Not examined: actual Win32 `CreateProcess`/`call` resolution of `\\` inside a quoted path on a real Windows host — untriable here, stays `asserted`.

- **Missing symlink resolution vs. the `sh` script's `readlink_f` (`iex:26-37`).** Rejected: I read `iex`; its `readlink_f` (`iex:26-34`) exists because a POSIX launcher may be reached through a symlink on `PATH`. The batch locates its sibling via `%~dp0`, the directory of the *running* `.bat`; the Windows launcher has no symlink-following requirement, so the absence is not a dropped guard but a platform difference. Status: `verified` for the callgraph claim (both files read); the "Windows has no symlink-resolution need" judgment is `asserted` (I cannot exercise Windows symlink/`.lnk` semantics here). Axes examined: source presence/absence of the resolver in both scripts. Not examined: behaviour when `iex.bat` is itself invoked through an NTFS symlink or junction — untriable in this sandbox.

- **`%*` re-passing the full original argv after the leading flags (`iex.bat:29`).** Rejected: intended construction — `--no-halt --erl "-user elixir" +iex` are prepended and the user's arguments follow verbatim via `%*`, mirroring `+iex "$@"` in `iex:38`. cmd.exe `%*` is unaffected by `shift` (and `iex.bat` performs no `shift`), so nothing is dropped or duplicated; the receiving parser in `elixir.bat` consumes the three prepended tokens and drops the first user positional to `:run`. Status: delegation side `verified` (read `elixir.bat:102,114,131,134`); the "cmd.exe `%*` unaffected by shift" property is `asserted`. Axes examined: which tokens the target parser consumes. Not examined: live cmd.exe `%*` expansion — untriable here.

- **Bare `iex` (no arguments) failing to reach the shell.** Rejected: with `%1` empty, `iex.bat:8-11` all compare false and fall to `:run`; `elixir.bat` then receives `--no-halt --erl "-user elixir" +iex` (so its `%1` is `--no-halt`, non-empty — `elixir.bat:9` documentation guard requires *both* `%1` and `%2` empty, so it is skipped), the loop sets `useIEx`, and `erl.exe` launches with `-noshell … -extra` — matching the sh script's no-`"$@"` path. Status: `verified` (read `elixir.bat:9,136-145`). Axes examined: the target's empty-arg documentation guard and `useIEx` gating. Not examined: none for the delegation logic; the underlying cmd.exe empty-`%1` comparison remains `asserted` as a platform primitive.

## Did I verify this?

- **Read this pass:** `iex.bat` (target file), `elixir.bat` (delegation target), `iex` (POSIX sibling). All three read in full.
- **Verified by reading:** the delegation target and argument arity (`elixir.bat:102,114,117-119,131,134-137`); the `--help` stdout/exit-0 vs stderr/exit-1 divergence (`iex.bat:14-26` vs `iex:9-24`); the empty-argv path (`elixir.bat:9`). These are no longer assumptions.
- **Could NOT verify by execution:** every cmd.exe-tokenisation axis (`%1` quote preservation, `""%1""` `IF` splitting, `\\` path collapse, `%*`-vs-`shift`, Windows symlink resolution). The sandbox has no `cmd.exe`; only POSIX `sh` is available, and reading a neighbour module does not substitute for pushing a value through a real interpreter. Per the re-review rule, these axes stay on `Not examined` and their entries stay `asserted` — none were promoted to `verified`.

## Upgraded from hedged assumption in first pass

- The first-pass caveat "the actual switch parsing … lives in `elixir.bat` (lines 117-119, etc.) and is out of scope for this file" was a hedged assumption. It is now **verified**: I read `elixir.bat` and confirmed `--remsh`/`--dot-iex`/`--dbg` at lines 117-119 and the three prepended flags at lines 102/114/131, including the ordering that preserves the `+iex → useIEx` invariant. No finding results; the delegation is correct.

## Dropped in re-review

None. Every prior finding and dismissal is preserved; the two "out of scope"/assumed items were upgraded to verified rather than dropped, and no dismissal was promoted from `asserted` to `verified` without executed evidence.
