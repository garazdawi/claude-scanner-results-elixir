# Cluster cross-file findings

No cluster cross-file findings

---

I read both cluster files (`iex`, `iex.bat`) plus the delegates they invoke (`elixir`, `elixir.bat`) and the sibling wrappers (`elixirc`, `elixirc.bat`) to check the sh↔bat parity invariant the threat model flags.

`iex` and `iex.bat` are faithful mirrors: both inject the identical switch sequence `--no-halt --erl "-user elixir" +iex` ahead of the user's arguments, both forward *all* original arguments to their respective delegate, and both gate only on the first argument for `--help`/`-h`. The divergences that do exist between them do not clear the bar for a cross-file security finding:

- **`"$@"` (sh) vs `%*` (bat) argument forwarding** (`iex:38`, `iex.bat:29`) — a real fidelity difference, but the injection risk of `%*` is a single-file property of `iex.bat` that a per-file reviewer surfaces on its own, and the dangerous re-parsing actually lives downstream in `elixir.bat` (delayed expansion, `set par="!par:"=\"!"`), reached identically by `elixirc.bat`/`mix.bat`/direct invocation — so it belongs to the `elixir`/`elixir.bat` cluster, not this one.
- **Help flags** — `iex.bat` also treats `/h` and `/?` as help (`iex.bat:10-11`) while `iex` does not; non-security.
- **Path resolution** — `iex` symlink-resolves via `readlink_f` (`iex:36`) while `iex.bat` uses literal `%~dp0` (`iex.bat:29`); not a clear vuln, and `elixir`/`elixir.bat` independently recompute their own `SCRIPT_PATH` for the `-pa .../ebin` code path regardless.

The injected `--erl "-user elixir"` is a fixed literal (no taint), and neither file contains any validation, auth check, or secret handling that the other bypasses. The substantive parity divergences in this launcher suite (e.g. erl-level switches leaking past `-extra` via unshifted `%*` on Windows) are between `elixir` and `elixir.bat`, which are outside this cluster.
