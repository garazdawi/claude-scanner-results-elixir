# /Users/lukas/code/claude-scanner/elixir/bin/mix.ps1
<!-- sha256: 67b1a63d7d8c5fcf0438e90c0170f152fbba6493795e1386703eb2005513ebcc -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T07:43:33Z_

## Summary

The launcher resolves its own and `mix.bat`'s path with `Get-ChildItem`, which interprets `[`/`]` in the install path as wildcard patterns and can fail to locate `mix.bat`, preventing Mix from starting.

## Bugfixes

#### `CSCAN-mix-getchilditem-wildcard-path`

- **Finding ID:** `CSCAN-mix-getchilditem-wildcard-path`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `mix.ps1:6` (top-level script body, `$mixBatPath` resolution)
- **Class:** Path assumptions (Windows-specific path resolution)
- **Impact:** A Windows user whose Elixir install path (or any parent directory, e.g. a home directory like `C:\Users\a[b]\...`) contains the literal characters `[` or `]` — both legal in NTFS names — cannot start Mix at all. `Get-ChildItem`'s positional `-Path` parameter treats its argument as a wildcard, so the bracketed segment is interpreted as a character-class pattern instead of a literal path. Both the outer call (`Get-ChildItem $MyInvocation.MyCommand.Path`) and the inner call (`Get-ChildItem (... + '\mix.bat')`) are affected; when the pattern fails to match, `$mixBatPath` is `$null`/errors and `& $mixBatPath $newArgs` on line 27 fails, so `mix`, `mix deps.get`, `mix test`, etc. all abort before the runtime launches.
- **Evidence:** asserted. No `pwsh`/`powershell` is available in the sandbox (`command -v pwsh powershell` → exit 1), so I could not execute the fragment. The mechanism is documented PowerShell behavior: `Get-ChildItem`'s `-Path` is wildcard-expanded and `-LiteralPath` exists specifically to suppress it; a path segment such as `a[b]` is parsed as a single-character class matching `a` followed by `b`, which does not match the literal directory `a[b]`.
- **Suggested fix:** Avoid wildcard-sensitive resolution entirely, e.g. build the path from the script's own directory without `Get-ChildItem`:
  ```powershell
  $mixBatPath = Join-Path (Split-Path -LiteralPath $MyInvocation.MyCommand.Path -Parent) 'mix.bat'
  ```
  (or, if a `FileInfo`/existence check is desired, use `Get-Item -LiteralPath`).

## Risks / caveats

- **Could not execute.** No PowerShell interpreter is present in the sandbox, so every claim here is reasoned from documented PowerShell semantics rather than reproduced. The wildcard behavior of `Get-ChildItem -Path` and the `-LiteralPath` remedy are well-established, but the exact failure surface (silent `$null` vs. terminating error) depends on the caller's `$ErrorActionPreference`.
- **Embedded-quote passthrough.** `& $mixBatPath $newArgs` (line 27) inherits PowerShell's historically fragile native-argument quoting. Simple spaces (e.g. `mix run -e "IO.puts 1"`) are quoted correctly, but arguments containing embedded `"` may be mangled before reaching `mix.bat`. This is a general PowerShell/native-call limitation rather than a defect introduced by this script, so it is noted rather than filed.
- **Spaces alone are safe here.** Unlike the POSIX launchers, spaces in the install path do *not* break this script: the path flows through single string variables and `& <FileInfo>`, both of which preserve spaces. Only wildcard metacharacters (`[`, `]`, and in principle `*`/`?`, which are illegal in NTFS names) are the hazard.

### Ruled out

- **Comma re-introduction loop (lines 9–24) mangling `mix do a, b`.** The loop appends a comma to every array element except the last (`$j -lt $args[$i].length - 1`, then `+= $args[$i][-1]`), reproducing the exact token stream cmd.exe would have delivered (`do`, `compile,`, `test`). This is the intended purpose of the shim (PowerShell splits comma arguments into arrays; the loop rebuilds them), and the loop bound is correct with no off-by-one — the last element is appended once, comma-free. Not a defect.
- **Empty `$newArgs` when invoked with no arguments.** The loop is skipped and `& $mixBatPath @()` calls `mix.bat` with no args, matching the documented bare `mix` invocation. Correct.

