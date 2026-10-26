# Folder correctness/bug profile: `elixir/bin`

## What this code actually is

**This folder contains no `.erl` files and implements no wire protocol.** It is the Elixir toolchain's **launcher layer**: POSIX `sh` scripts (`elixir`, `elixirc`, `iex`), Windows batch files (`elixir.bat`, `elixirc.bat`, `iex.bat`, `mix.bat`), a PowerShell shim (`mix.ps1`), and an `escript`-style Elixir stub (`mix`, whose only body is `Mix.CLI.main()`). Their sole job is to **parse a command line and re-exec `erl`/`erl.exe`/`run_erl`** with a reconstructed argument vector. Version stamped `ELIXIR_VERSION=1.21.0-dev`. No RFC/NIST/standards body governs this — the relevant "spec" is the self-documented usage text in each script and POSIX `sh` / cmd.exe / PowerShell quoting semantics. The per-file reviewer should judge these as **argument-marshalling shell programs**, not Erlang modules; the protocol-decoder / gen_server / supervisor framing in the review brief does not apply here.

Public entry points: the executables themselves (`elixir [opts] file.exs`, `elixirc`, `iex`, `mix`). `elixirc`, `iex`, and `mix` are thin wrappers that all `exec`/`call` back into `elixir`(`.bat`), so **`elixir`/`elixir.bat` is the one file where real logic lives**; the others are delegation stubs.

## Trust / contract boundaries

- **Caller-supplied argv** (operator/local user) is the primary untrusted-shaped input. Flags like `--erl`, `--cookie`, `--name`, `--boot`, `--vm-args`, `--erl-config`, `--pipe-to`, `-e`, `--rpc-eval` all flow into the re-exec'd command line.
- **Environment variables** are an implicit input surface: `ELIXIR_ERL_OPTIONS`, `ELIXIR_CLI_DRY_RUN`, `OSTYPE`, `ERTS_BIN`, `PATH` (for locating `erl`). `ELIXIR_ERL_OPTIONS` is injected **unquoted** into the final `set --`/command line by design.
- **Contract the caller must honour:** "standalone" options (`-v`, `--help`, `--short-version`) must not be combined with others; flags marked `(*)` take a fixed arg count (the `C=`/`S=` counters in `elixir` and the `shift` counts in `.bat` encode this). The `--pipe-to` path is **Unix-only** (batch explicitly rejects it).
- These launchers are typically fed **operator-controlled** argv, not network data. The realistic threat model is a *malformed or adversarial argv/env* (e.g. supplied by a wrapping script, CI config, or a `mix` task), not a remote peer.

## Failure-mode model (what to actually look for)

These are shell/batch bug classes, not Erlang ones:

- **Argument-count desync.** In `elixir`, each flag sets `C` (copy N words to the `-extra` tail) or `S` (shift N, consume as Erlang args). An off-by-one in `C`/`S` for any flag silently drops or duplicates a following argument, or shifts a value into the wrong bucket. This is the single most likely correctness bug in the file. Cross-check every `case` arm's `C=`/`S=` against the flag's real arity.
- **Quoting / word-splitting.** `ERL="$ERL $2"` accumulates into a **space-joined string** that is later re-split by the shell — any arg containing spaces breaks. Contrast with the `erl_set`/`E` "array emulation" used for `--cookie`, `--name`, etc., which *does* preserve spaces. A flag routed through `ERL` instead of `erl_set` when it can carry spaces is a latent bug.
- **`starts_with` misuse in `--pipe-to`:** `[ "$(starts_with …)" ]` tests whether the *command substitution output* is non-empty. `starts_with` prints nothing (it only sets exit status via `true`/`false`), so the guard is effectively always false — the switch-rejection check likely never fires. Worth flagging.
- **`readlink_f`** recursive symlink resolver: no cycle guard; a symlink loop recurses until the shell dies. Also `cd … > /dev/null` relies on `set -e` staying armed.
- **`.bat` delayed-expansion quoting:** `set par="!par:"=\"!"` re-quotes embedded quotes; the `if ""==!par:-e=!` substring tests match *any* arg containing the flag text, not just the flag — a value like `-example` can be misclassified. cmd.exe `shift` does not affect `%*`, so `%*` at the end re-passes the *original* argv regardless of prior `shift`s (intentional here, but a classic trap).
- **PowerShell array flattening** in `mix.ps1` reintroduces commas manually; an off-by-one in the inner `for ($j … length-1)` loop would drop or mis-place a comma.

## Cross-file invariants the per-file reviewer would miss

- `elixirc`, `iex`, `mix.bat`, `elixirc.bat`, `iex.bat` **all delegate to `elixir`/`elixir.bat`**. Any argv the wrappers prepend (`+elixirc`, `+iex`, `--no-halt --erl "-user elixir"`, the `mix` script path) becomes `$1…` inside `elixir`'s parser and **must be recognised by a `case`/`if` arm there**. A reviewer inspecting `iex` alone cannot tell whether `+iex`/`--dbg`/`--remsh` are handled — that lives in `elixir`. `+elixirc` and `+iex` are consumed (`C=1`) in `elixir`; `mix` is an Elixir file passed as the script path, so it falls into the `*)` "rest are program args" branch.
- The `+iex` marker set in the wrapper is the **only** thing that flips `MODE=iex`, which in turn decides whether `-s elixir start_cli` is prepended. If the parser's `*)` catch-all fires *before* `+iex` is seen (i.e. argument order), `MODE` stays `cli` — an ordering invariant the wrappers rely on by always putting `+iex` first.
- The `sh` and `.bat`/`.ps1` variants are **intended to be behaviourally equivalent**; divergence between them (a flag handled in one, dropped or given a different arity in the other) is a real bug even though each file looks internally consistent. Diff the flag tables across `elixir` ↔ `elixir.bat` when assessing any single arm.
- `ERTS_BIN=` / `ERTS_BIN="$ERTS_BIN"` is a **release-rewrite anchor** — the comment forbids changing the second line. Edits there look harmless in isolation but break `mix release`-generated launchers.