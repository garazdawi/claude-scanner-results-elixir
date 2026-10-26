# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.build.ex
<!-- sha256: 430d2884f1fedb2ea18db4b4dc5f39cdfe7e0c916c732c083708b7d5d6aaeaf4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:36:15Z_

## Summary

`mix archive.build` (`run/1`) is a small, well-behaved task: option parsing, source/target resolution, and zip assembly all work correctly on the happy path and across the `-i`/`-o`/`--include-dot-files`/`--no-compile` variations I exercised. The only defect is that a failed `:zip.create` surfaces as an unhandled `MatchError` rather than a friendly `Mix.raise`.

## Bugfixes

#### `CSCAN-archive.build-zip-create-badmatch`
- **Finding ID:** `CSCAN-archive.build-zip-create-badmatch`
- **Severity:** Low
- **Confidence:** high
- **Location:** `elixir/lib/mix/lib/mix/tasks/archive.build.ex:122`, `create/3`
- **Class:** Error-handling asymmetry (unmatched `{:error, _}` → `MatchError`)
- **Impact:** When the output archive cannot be written (e.g. `-o` points into a non-writable directory), the hard match `{:ok, _} = :zip.create(...)` raises a raw `MatchError` with an internal stack trace instead of the task's usual `Mix.raise` friendly message. The build still fails, but the caller gets an implementation-leaking crash rather than an actionable "could not create archive" error, which is inconsistent with every other guarded failure in this task (`Mix.raise` at lines 90, 104, 108).
- **Evidence:** `verified`. In a project, `mix archive.build -o ro/out.ez --no-compile` with `ro` mode `555`:
  ```
      {:error, :eacces}

      (mix 1.21.0-dev) lib/mix/tasks/archive.build.ex:122: Mix.Tasks.Archive.Build.create/3
      (mix 1.21.0-dev) lib/mix/tasks/archive.build.ex:111: Mix.Tasks.Archive.Build.run/1
  ```
  (By contrast, a missing source directory is reported cleanly via `Mix.raise` at line 108.)
- **Suggested fix:** Match the result and raise a task-appropriate error, e.g.
  ```elixir
  case :zip.create(String.to_charlist(target_path), file_list) do
    {:ok, _} -> :ok
    {:error, reason} ->
      Mix.raise("Could not create archive #{inspect(target)}: #{inspect(reason)}")
  end
  ```

## Risks / caveats

- **`File.write`/`File.rm` return values ignored (lines 82, 84).** If writing the `.elixir` marker into `app_path` fails, the archive is produced silently without the Elixir version requirement (weakening the install-time version check). On the normal flow `app_path` exists and is writable, so I could not construct a realistic trigger; left as a robustness note, not a finding.
- **Original `args` forwarded to `compile` (line 70).** Archive-specific flags (`-o`, `-i`, `--include-dot-files`, etc.) are passed through to `Mix.Task.run(:compile, ...)`. I verified this is harmless: `Mix.Tasks.Compile` parses with a *non-strict* `OptionParser.parse` (`compile.ex:141`), so unknown switches are tolerated rather than raising. Compilation succeeded in every variant tested.
- I verified `Path.wildcard(".elixir")` does match the dotfile literal, that `Path.join(charlist_dir, string)` correctly produces the archive-internal path (charlist treated as chardata), and that `include_dot_files?` gates only the `priv` wildcard as documented. Review was dynamic (built-from-source `mix`) plus static reading of `Mix.Local.name_for/archive_name`; I did not audit `:zip.create`'s internals beyond its documented `{:ok,_}|{:error,_}` contract.

