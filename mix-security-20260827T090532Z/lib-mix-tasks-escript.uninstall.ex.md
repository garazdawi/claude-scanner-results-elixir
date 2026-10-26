# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/escript.uninstall.ex
<!-- sha256: 0bec2bece5e0935a17fcf0933dd3c5e4eccbfdf80d9e9a5e78dae6750fb4cef7 -->

_Analysed on 2026-08-27T11:03:10Z_

## Summary

`Mix.Tasks.Escript.Uninstall` is a thin wrapper whose only externally-influenced input is the operator-typed escript name in `argv`; no dependency-, network-, archive-, or file-content-derived data reaches its single `File.rm` sink, so there are no in-scope security findings.

## Security

No findings.

The entire file is one function:

```elixir
def run(argv) do
  if path = Mix.Local.Installer.uninstall(Mix.path_for(:escripts), "escript", argv, @switches) do
    File.rm(path <> ".bat")
  end
end
```

All uninstall logic lives in `Mix.Local.Installer.uninstall/4` (a different file, `lib/mix/local/installer.ex:301`). The only thing this file adds is deleting the Windows `.bat` companion of whatever path that function returns. `path` is derived solely from the escript **name** the operator types on the command line (`List.first(argv)`); there is no code path by which a dependency, git ref, archive entry, network response, or a file the project merely reads can influence `argv` or `path`. Under this codebase's threat model the operator is explicitly out of scope, and every candidate sink concern collapses to operator-controlled or local-co-tenant input.

### Ruled out

- **Path traversal via the escript name (`File.rm(path <> ".bat")`).** `uninstall/4` returns `Path.join(root, name)` with `name` from `argv`, so `name = "../../../../etc/passwd"` yields the sink argument `/etc/passwd.bat` (verified: `Path.join("/home/user/.mix/escripts", "../../../../etc/passwd") |> Kernel.<>(".bat") |> Path.expand` → `"/etc/passwd.bat"`). This is entirely operator-supplied CLI input — the victim would be typing the malicious name themselves — which the threat model places out of scope. No non-operator source can reach `argv`.
- **Crash / `nil` at `path <> ".bat"`.** The `if path = …` guard fires only when `uninstall/4` returns a binary (it returns `found`, a binary, or `nil`), so the string concatenation never sees a non-binary. `File.rm` is the non-raising form: verified `File.rm("/tmp/does-not-exist-xyz.bat")` → `{:error, :enoent}`, so a missing companion (the common case on non-Windows) is a silent no-op, not an exception.
- **TOCTOU between the installer's `File.rm_rf!(found)` and this file's `File.rm(found <> ".bat")`.** A racer swapping in a symlink cannot escalate: verified `File.rm` on a symlink removes the link itself, leaving the target intact (`target survives? → true`). Any racer would also need write access to the operator's own escripts directory (co-resident, operator-trusted code) — outside the remote-content attacker model.
- **Dangerous installer sinks** (`String.to_unsafe_atom` on package/app names, git URL → `System.cmd`, `Mix.Utils.read_path`, `Path.wildcard`). These exist in `installer.ex` but sit on the `install` path, not `uninstall`, and are not part of this file. The `uninstall/4` path this file calls uses only `File.exists?`, `Path.join`, `Path.wildcard(name <> "-*")`, and `File.rm_rf!` — all keyed on operator input.

## Risks / caveats

- I reviewed only this file; the real deletion (`File.rm_rf!`), name matching, and confirmation prompt live in `Mix.Local.Installer.uninstall/4`, which I read to complete the trace but did not audit as an in-scope target. If that shared helper is separately reviewed, the operator-controlled traversal noted above (`Path.join(root, name)` with no `Path.safe_relative` confinement, and `matching_package`'s `Path.wildcard(name <> "-*")`) is the item to weigh there — still gated on operator argv, but that is where the path is constructed.
- I did not drive the full `mix escript.uninstall` task end-to-end (it needs an installed escript fixture); I verified the individual runtime behaviors the conclusion depends on (`File.rm` non-raising, traversal resolution of `path <> ".bat"`, symlink-not-target deletion) directly with the source-built Elixir 1.21.0-dev / OTP 29 toolchain and quoted the outputs above.

