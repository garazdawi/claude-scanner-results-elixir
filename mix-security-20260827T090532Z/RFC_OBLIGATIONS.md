<!-- rfc-governed: no -->
<!-- standards: -->

# Compliance obligation list — `elixir/lib/mix`

## Summary

This folder is the source of Elixir's **Mix** build tool: task runner, dependency resolver/converger, SCM adapters (git/path), compilers (Elixir/Erlang/yecc/leex/app), release/escript/archive builders, config loaders, profilers, and shell abstractions. It is application and build-tooling code. Nothing here implements a network protocol, an on-the-wire message format, a negotiated state machine, or a cryptographic algorithm defined by an external protocol/algorithm standard. Where the code touches security-relevant behaviour (fetching deps over the network via external SCMs, unpacking archives/escripts, running shell commands, writing files), correctness is governed by ordinary secure-coding practice (CERT-C / CWE class issues, path/command injection, TOCTOU) — which the per-file security review already checks directly — not by an RFC/NIST protocol conformance gap.

Accordingly the RFC-governance verdict is **no** and the external-standards list is **empty**. There is no downstream protocol/algorithm compliance obligation to enumerate for this folder.

## Applicable standards / contracts

### External standards
None. No IETF RFC, ITU-T, W3C, ISO/IEC, NIST SP, FIPS, or consumed third-party wire/report format is implemented here. (Mix consumes de-facto formats such as Git's SCM behaviour, rebar/Hex metadata, and Zip archives, but this folder drives external tools / uses OTP's own libraries rather than implementing any standardized protocol or algorithm.)

### OTP / Elixir-internal contracts (per-file context, NOT compliance obligations)
These are guidance for the per-file reviewer only — do **not** treat them as standard-conformance obligations:

- **`Mix.Task` / `Mix.Task.Compiler` behaviours** — `run/1`, `manifests/0`, `clean/0` callback shapes and their documented return conventions.
- **`Mix.SCM` behaviour** (`lib/mix/scm.ex`, `dep/*`) — `fetchable?/0`, `checkout/1`, `update/1`, `lock_status/1`, `equal?/2` callback contracts governing dependency fetch/lock semantics.
- **`Mix.Shell` behaviour** (`shell.ex`, `shell/*`) — `info/1`, `error/1`, `prompt/1`, `cmd/2`, `yes?/2` contracts.
- **`Mix.RemoteConverger` behaviour** (`remote_converger.ex`) — the Hex integration hook contract.
- **Dependency lockfile / manifest shapes** — `dep/lock.ex` and compiler manifests are internal serialization formats consumed only by Mix itself, versioned internally; not an external standard.
- General `-spec`/`@doc` return shapes throughout.

## Checkable sub-claims

None applicable to a protocol/algorithm compliance pass. This folder has no MUST/MUST-NOT/SHOULD-NOT clauses derived from an external protocol, wire format, or cryptographic algorithm standard. Security-relevant behaviours in these files (dependency fetching, archive/escript extraction, config evaluation, shell command execution, file writes under build paths) are already covered by the per-file security review as secure-coding / CWE-class concerns (command injection, path traversal, unsafe extraction, TOCTOU, arbitrary code execution from untrusted project config) and require no separate RFC/NIST conformance list.

## Per-file checking instructions

Because the folder is not RFC-governed and the external-standards list is empty, the per-file reviewer should **not** look for protocol/algorithm conformance gaps here and should not manufacture RFC/NIST findings. Review each file under the ordinary secure-coding and correctness rubric instead: injection (shell/command/path), unsafe archive or escript extraction, untrusted-input evaluation (config, lockfiles, rebar configs), TOCTOU on build artifacts, and behaviour-callback contract violations. Treat the OTP/Elixir-internal contracts above only as context for judging whether a task/SCM/shell/compiler implementation honors its documented callback shape — not as external compliance obligations.