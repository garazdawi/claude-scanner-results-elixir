# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/escript.ex
<!-- sha256: f92fad20124595d28df9ceea5189999c9944cdd57ae3c88b9f6bc614bf4bcac4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:51:55Z_

Confirmed. `File.ls` returns the broken symlink entry, and `File.stat!` then raises. Let me write up the review.

## Summary

`escript.ex` implements `mix escript`, which lists installed escripts under `~/.mix/escripts`; the file is small and mostly correct, but its filter step uses the bang `File.stat!/1` on every directory entry, which crashes the task on a dangling symlink or a file removed mid-listing.

## Bugfixes

#### `CSCAN-escript-stat-bang-crash`
- **Finding ID:** `CSCAN-escript-stat-bang-crash`
- **Severity:** Low
- **Confidence:** high
- **Location:** `escript.ex:40` (`executable?/1`), reached from `escript.ex:25` in `run/1`
- **Class:** Error-handling asymmetry
- **Impact:** Running `mix escript` crashes with an unhandled `File.Error` (`could not read file stats … no such file or directory`) if the `~/.mix/escripts` directory contains a broken symlink, or if any entry is removed by another process between the `File.ls` listing and the per-file `File.stat!`. The task cannot list *any* escripts as long as one bad entry is present, and the error message points at file stats rather than telling the user their escripts dir has a stale link. The surrounding code deliberately tolerates a missing directory (`list_dir` swallows the `File.ls` error to `[]`), so the bang here is an inconsistent, unintended failure mode rather than a designed contract.
- **Evidence:** `verified`. With a broken symlink in a directory:
  ```
  $ elixir -e 'IO.inspect(File.ls("/tmp/esctest")); File.stat!("/tmp/esctest/brokenlink")'
  {:ok, ["brokenlink"]}
  ** (File.Error) could not read file stats "/tmp/esctest/brokenlink": no such file or directory
  ```
  `File.ls` returns the entry, and `File.stat!` — invoked for that same entry inside `executable?/1` — raises. `run/1` has no `rescue`, so the whole task aborts.
- **Suggested fix:** Replace `File.stat!(path)` with a non-raising form and treat unreadable entries as non-executable, e.g.:
  ```elixir
  defp executable?(path) do
    case File.stat(path) do
      {:ok, stat} -> executable_stat?(path, stat)
      {:error, _} -> false
    end
  end
  ```
  (moving the existing `:os.type/0` logic into `executable_stat?/2`).

## Risks / caveats

- The file has no doctests and no `@spec`s, so there is nothing to check for documentation/spec mismatch beyond the `@moduledoc` (which accurately describes "lists installed escripts" and the install path).
- The win32 branch (`File.exists?(path <> ".bat")`) was reasoned about, not executed (no Windows sandbox); it appears correct — it keeps base script files and filters `.bat` companions — but this is `asserted` for that platform.
- Severity is rated Low because the affected task is read-only and informational; the crash is a nuisance and a poor error message rather than data loss. It is nonetheless reachable through the documented public entry point (`mix escript` / `run/1`) via ordinary filesystem state a user can produce without malice.

