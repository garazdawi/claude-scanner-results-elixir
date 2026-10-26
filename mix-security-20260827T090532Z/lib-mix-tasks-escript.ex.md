# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/escript.ex
<!-- sha256: f92fad20124595d28df9ceea5189999c9944cdd57ae3c88b9f6bc614bf4bcac4 -->

_Analysed on 2026-08-27T11:00:25Z_

I have what I need. Both candidate sinks are confirmed empirically. Let me record the key reachability facts before writing up.

- `print/1` (line 58) emits each filename via `Mix.shell().info(["* ", item])` with **no neutralization** — verified that a filename containing `\x1b[2K\r` reaches the terminal as raw `ESC`/`CR` bytes.
- `executable?/1` (line 40) calls `File.stat!/2`, which **raises and aborts the whole task** on a dangling symlink or a file removed between `File.ls` and the stat — verified `File.Error` crash.
- Reachability of a hostile filename: `escript.install` writes the basename from `Path.basename(URI.parse(src).path)` (raw control bytes survive parsing) or, for source builds, from the untrusted project's declared escript `:name` (verified `name_for` returns `"evil\e[2Kname"` verbatim).

## Summary

`Mix.Tasks.Escript` is a small read-only listing task whose only externally-influenced data is the set of filenames in `~/.mix/escripts`; it prints those filenames to the terminal without neutralizing control characters, and crashes the whole task if any entry is a dangling symlink.

## Security

#### `CSCAN-escript-filename-terminal-escape`
- **Finding ID** — `CSCAN-escript-filename-terminal-escape`
- **Severity** — Low
- **Confidence** — high
- **Location** — `elixir/lib/mix/lib/mix/tasks/escript.ex:58`, `print/1` (`Mix.shell().info(["* ", item])`)
- **Class** — Template / interpolation (improper neutralization of escape sequences in terminal output, CWE-150)
- **Impact** — `mix escript` writes each installed-escript filename to the terminal verbatim. A filename containing ANSI/terminal control sequences is emitted raw, so an attacker who controls a filename in `~/.mix/escripts` can erase/rewrite lines, hide the real entry, spoof other tool output, or (on permissive terminals) drive clipboard/title/response-injection behaviour — all triggered merely by a victim running the benign listing command, without ever executing the escript. Filenames there are influenceable through `escript.install`: for a prebuilt URL install the name is `Path.basename(URI.parse(src).path)`, which preserves raw control bytes (verified: `URI.parse("https://host/a\eb\rc").path` → basename `"a\eb\rc"`); for a source build the name is the untrusted project's declared escript `:name` (verified: `Mix.Local.name_for(:escripts, escript: [name: "evil\e[2Kname"])` → `"evil\e[2Kname"`). This is the same class GNU `ls` hardened against by quoting non-printable filenames on a TTY. The bounding factor (and why Low) is that landing a hostile filename in the trusted `~/.mix/escripts` is a meaningful precondition and impact is terminal-dependent.
- **Evidence** — `verified`. Placed a file named `real\x1b[2K\rINJECTED-owned-your-terminal` (exec bit set) in the escripts dir and ran the real task:
  ```
  $ MIX_HOME=/tmp/mixhome_test mix escript 2>&1 | cat -v
  * real^[[2K^MINJECTED-owned-your-terminal
  * normaltool
  Escripts installed at: /tmp/mixhome_test/escripts
  ```
  `cat -v` shows the raw `^[[2K^M` (ESC `[2K` = erase-line, CR); on a real TTY this hides the `real` prefix and leaves only the injected text.
- **Suggested fix** — Neutralize control characters before printing, e.g. replace each item with an escaped form (`String.replace(item, ~r/[\x00-\x1f\x7f]/, ...)` or `inspect(item)`) inside `print/1`, matching how the code already uses `inspect/1` for user-facing paths elsewhere in the installer.

#### `CSCAN-escript-stat-crash-on-listing`
- **Finding ID** — `CSCAN-escript-stat-crash-on-listing`
- **Severity** — Low
- **Confidence** — high
- **Location** — `elixir/lib/mix/lib/mix/tasks/escript.ex:40`, `executable?/1` (`stat = File.stat!(path)`)
- **Class** — Concurrency / resource (unhandled error → availability; TOCTOU on the `File.ls`→`File.stat!` window)
- **Impact** — `run/1` lists the directory and then calls `File.stat!/2` on every entry. `File.stat!` follows symlinks and raises `File.Error` when the target cannot be stat'd, so a single **dangling symlink** in `~/.mix/escripts` — or a plain file deleted in the race window between `File.ls` and `File.stat!` — aborts the entire `mix escript` command with an exception instead of skipping the entry. This only denies the listing itself (the user can fall back to `ls`), the trust boundary is weak (needs write access to the escripts dir), and it is self-inflicted, hence Low.
- **Evidence** — `verified`. With a broken symlink present:
  ```
  $ MIX_HOME=/tmp/mixhome_test2 mix escript
  ** (File.Error) could not read file stats ".../escripts/brokenlink": no such file or directory
      (elixir) lib/file.ex:517: File.stat!/2
      (mix) lib/mix/tasks/escript.ex:40: Mix.Tasks.Escript.executable?/1
      (mix) lib/mix/tasks/escript.ex:25: Mix.Tasks.Escript.run/1
  ```
- **Suggested fix** — Use the non-raising `File.stat/2` and treat `{:error, _}` as “not executable” (return `false`), so unreadable/vanished entries are skipped rather than aborting the listing.

### Ruled out
- **Path traversal via `Path.join(escripts_path, filename)` (line 25):** `filename` comes from `File.ls`, whose directory entries never contain `/`, `.`, or `..`, so no component escapes the escripts root; and the joined path only feeds `stat`, not any write/delete.
- **Code execution / atom creation / command execution / deserialization / network:** none present — this file only stats and prints; there is no `eval`/`apply`, no `String.to_atom`/`Module.concat`, no `System.cmd`/shell, no `binary_to_term`, no outbound request.
- **`"...#{Mix.path_for(:escripts)}"` interpolation (line 59):** the path is derived from operator-controlled `MIX_HOME`/defaults, not from any attacker-crossable boundary.
- **`Mix.shell()` dispatch:** the shell module comes from operator Mix config, not external input.

## Risks / caveats
- Both findings hinge on getting a hostile filename (or dangling symlink) into `~/.mix/escripts`. The most realistic delivery is `mix escript.install` from an attacker-supplied URL/source; for **git/github/hex source installs** the project code already runs during the build, so terminal injection there is strictly weaker than the RCE the operator has already accepted — the escalating case is a **prebuilt URL/local install** (no build-time code execution) or any co-resident process able to write to the escripts dir.
- I verified sink behaviour against the installed Elixir 1.21.0-dev / OTP 29 build (crash stack points at `escript.ex:40`/`:25`, matching the file under review); I did not stand up a full end-to-end `escript.install` from a hostile HTTP server to confirm a control-char basename survives the interactive “Do you trust…?” prompt — that prompt renders `inspect(src)` (escaped), so a careful operator could notice the escapes there even though the later `mix escript` output does not escape them.
- No attempt was made to assess non-security stylistic aspects; per instructions only security-relevant behaviour was reviewed.

