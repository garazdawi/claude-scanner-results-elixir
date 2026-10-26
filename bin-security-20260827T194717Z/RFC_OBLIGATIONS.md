<!-- rfc-governed: no -->
<!-- standards: -->

# Standards obligations — `elixir/bin`

This folder contains the Elixir toolchain launcher scripts: POSIX `sh` scripts (`elixir`, `elixirc`, `iex`, `mix`), their Windows `.bat` counterparts, and a PowerShell wrapper (`mix.ps1`). There are **no `.erl` / `.hrl` files** here and no protocol, wire-format, or cryptographic-algorithm code. Each script parses command-line switches, sets environment variables, assembles an `erl`/`erlexec` invocation, and `exec`s the runtime. Their correctness is governed only by shell-scripting robustness and OS-API semantics, not by any external protocol/algorithm standard.

## Applicable standards / contracts

### External standards
None. No IETF RFC, ITU-T, W3C, ISO/IEC, NIST, or FIPS standard governs these launcher scripts, and no de-facto external wire/report format is produced. The `<!-- standards: -->` marker is intentionally empty.

The only arguably-external references are general shell/OS conventions — POSIX.1 / IEEE 1003.1 shell and utility semantics (word splitting, `exec`, quoting, `set -e`) — but these are secure-coding / OS-API concerns, not a protocol or algorithm specification. Per the scan instructions they are already covered by the per-file secure-coding review and do **not** warrant an `rfc-governed: yes` verdict.

### OTP-internal contracts (per-file context, NOT compliance obligations)
None. These are shell/batch/PowerShell scripts, not Erlang modules; there are no `-spec`/`@doc` shapes or behaviour callbacks to flag.

## Checkable sub-claims

No external protocol/algorithm obligations apply, so there is no standards-conformance list to enumerate. The per-file security review already covers the relevant failure modes for launcher scripts (shell word-splitting / injection via unquoted expansions, `IFS` handling, unsafe `eval`, argument-forwarding correctness, environment-variable trust). Those are secure-coding checks, not RFC/NIST clauses, and belong to that review rather than this list.

## Per-file checking instructions

Treat this folder as out of scope for the RFC/NIST compliance pass: there are no external protocol/algorithm sub-claims to match against these files. The per-file reviewer should fall back to its normal secure-coding pass (shell-quoting, injection, and argument/environment-handling classes) and should not report gaps for any RFC/NIST/CIS obligation, since none governs these launcher scripts.