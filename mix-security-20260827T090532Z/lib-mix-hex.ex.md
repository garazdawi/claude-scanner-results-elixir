# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/hex.ex
<!-- sha256: 431728cc068f7b38d250c9dcaca5c0393bf152876331a21a99b2b4f3c8b0ece3 -->

_Analysed on 2026-08-27T09:43:15Z_

Verified. `url/0` reads `HEX_BUILDS_URL` (precedence) then `HEX_MIRROR`, else the hardcoded `https://builds.hex.pm` default — all environment-controlled, and a plaintext `HEX_MIRROR` is passed through unvalidated. `Version.match?` raises on malformed input, but its input is the trusted installed-Hex version. That's the complete externally-influenced surface of this file. Here is the report.

## Summary
`Mix.Hex` is a thin shim around the optional `Hex` package manager (load/version check, install prompt, start, and a mirror-URL accessor); its only externally-influenced value is an operator-set environment variable, and it contains no security findings.

## Security
No findings.

### Ruled out
- **`url/0` returns an unvalidated, operator-controlled mirror URL (`hex.ex:79-81`).** `HEX_BUILDS_URL || HEX_MIRROR || "https://builds.hex.pm"`. The value has no scheme/TLS check, and a `HEX_MIRROR=http://…` is passed through verbatim (verified: `url/0` returned `"http://evil.example"`). But the input is an **environment variable** — operator-supplied config, explicitly out of scope in the folder threat model (the operator already controls the build). The `HEX_MIRROR` mechanism is Hex's documented mirror feature. The actual network fetch happens in the consumers (`local.hex.ex:73-88`, `local.rebar.ex:114-126`), which gate the download with `--sha512`; that the checksum originates from the same mirror is a property of those files and of Hex's mirror design, not a defect introduced here.
- **`Version.match?(Hex.version(), @hex_requirement)` crash on malformed version (`hex.ex:44`).** Verified that `Version.match?` raises `Version.InvalidVersionError` on garbage input, but the argument is `Hex.version()` from the already-installed, trusted Hex package (`@hex_requirement` is a hardcoded literal). Only reachable via a maliciously-crafted Hex module, which is the "installed archive runs arbitrary code" scenario the threat model places out of scope.
- **Interpolation of `Hex.version()` into `shell.info` (`hex.ex:45`).** A terminal-escape/format-injection sink only if `Hex.version()` is hostile; that source is the trusted package manager, not a dependency/network peer/archive. Not attacker-reachable.
- **`Mix.Tasks.Local.Hex.run(["--force"])` triggers a network install of Hex (`hex.ex:20,48`).** Argument list is a hardcoded literal — no argument/option injection. Installing Hex from the operator-configured source on operator confirmation (or `--force`) is the documented, intended behavior.
- **Code-execution / atom / command / path / archive / deserialization sinks.** None present in this file: no `Code.eval*`, no `String.to_atom`/`Module.concat`, no `System.cmd`/`os.cmd`/shell, no `File`/`Path`/`:zip`/`:erl_tar`, no `binary_to_term`, no crypto comparisons.

## Risks / caveats
- The end-to-end integrity of a mirror-served Hex/Rebar install (whether `--sha512` meaningfully protects when the checksum list is fetched from the same operator-chosen mirror) lives in `Mix.Local.find_matching_versions!`, `Mix.Tasks.Archive.Install`, and `Mix.Utils.read_path`, not in this file; I did not audit those here beyond confirming the call shape, and the mirror URL itself is operator-controlled either way.
- I treated the `Hex` module (`Hex.version/0`, `Hex.start/0`) as trusted infrastructure. A compromised installed Hex archive could feed hostile data to lines 44-45, but that is the out-of-scope "operator installed evil code" boundary, not a flaw in `hex.ex`.

