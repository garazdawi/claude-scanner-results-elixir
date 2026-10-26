# Cluster cross-file findings

No cluster cross-file findings

---

I read all three cluster files (`mix`, `mix.bat`, `mix.ps1`), the delegates they invoke (`elixir.bat`, and via shebang the sh `elixir`), and the sibling wrappers, then checked each cross-file security pattern against the argv-encoding contract the threat model flags between these files. Nothing clears the bar for a *new* cross-file security finding:

- **`mix.ps1` → `mix.bat` comma-rejoining contract (`mix.ps1:9-24,27` → `mix.bat:7`).** The loop that "reintroduces" commas appends `,` to all-but-last array elements and forwards them as *separate* `$newArgs` elements, so `mix.ps1 a,b,c` reaches `mix.bat` as three whitespace-separated args (`a,` `b,` `c`) instead of the single token `a,b,c`. This is a genuine cross-file argv-encoding divergence, but it is **non-security and already covered** by the per-file pass (`mix.ps1.md:28`, and logged in `CROSS_FILE.md:24`): the transform prepends no `-` (no switch injection) and introduces no `cmd.exe` metacharacter, and the array branch only triggers on *unquoted* commas typed by whoever writes the PowerShell command line (the trust root) — a wrapper forwarding a proper string array is untouched by it.

- **BatBadBut metacharacter passthrough (`mix.ps1:27`, `mix.bat:7`).** Both files forward caller argv verbatim into the `cmd.exe`/batch re-parse chain. This is the same chain reported *per-file* in both `mix.bat.md` (`CSCAN-mix-batch-argv-cmd-injection`) and `mix.ps1.md` (`CSCAN-mix-ps1-batch-arg-injection`); the actual re-parse sink lives downstream in `elixir.bat` (delayed expansion, unquoted `%*` at `elixir.bat:145`), outside this cluster and owned by that cluster's review. No cluster file re-validates, but no cluster file is the sink either, so there is no A-validates/B-skips split to report here.

- **Switch-injection invariant holds (`mix.bat:7` → `elixir.bat:93-147`).** `mix.bat` inserts `"%~dp0\mix"` as the first positional token, which matches no branch in `elixir.bat`'s parse loop and falls straight through to `:run`, so every forwarded user arg lands in `-extra` (Elixir `System.argv`) and none can occupy an `erl` switch position (`-setcookie`, `-config`, `-boot`, `-args_file`). Verified by reading the parser; parity with the Unix `mix` shebang path (`*)` case in `elixir:190-197`) holds.

- **No validation, auth, secret, or memory-management state is shared across these three files** — there is nothing for a trust-boundary/authz/crypto split or a use-after-free to hinge on. `--cookie`/`--name` (the distributed-Erlang auth surface) are handled entirely in `elixir`/`elixir.bat`, not reachable through any of the mix wrappers as a switch.

The one cross-file property the threat model singled out for this cluster — `mix.ps1`'s comma-rejoining reconstructing what `mix.bat`/`elixir.bat` expect — is broken, but it is a correctness defect already surfaced by the single-file pass and carries no security consequence, so it is not re-reported here.
