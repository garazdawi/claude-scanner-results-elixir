# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/converger.ex
<!-- sha256: 25cf3c6294021362afd72dd90c0c5bf7287f02aba6df4a0eef67412564879ae8 -->

_Analysed on 2026-08-27T09:30:52Z_

I have completed a full per-sink review of the file and verified the only atom-creating call is bounded. Here is my report.

## Summary
`Mix.Dep.Converger` is pure dependency-graph convergence and merge logic operating on already-parsed `Mix.Dep` structs, and after tracing every sink I found no security vulnerability reachable through its public API.

## Security

No findings.

### Ruled out

- **`String.to_unsafe_atom("diverged#{key}")` (lines 349, 353) — atom-table exhaustion (CWE-770).** This is the atom-*creating* variant (`:erlang.binary_to_atom/2`), so it warrants scrutiny, but its only interpolated component is `key`, which `with_matching_only_and_targets/3` supplies as the literal `:only` or `:targets` (grep confirms `with_matching/5` has no other callers — lines 326–327 are the sole call sites). `verified`: driving both reachable inputs produces exactly `:divergedonly` and `:divergedtargets` — a fixed two-element set, not attacker-influenced. No unbounded atom creation.
- **Behaviour/module dispatch `remote.converge/deps/remote?/post_converge` (75, 108, 118, 143) and `scm1.equal?(opts1, opts2)` (383).** `remote` is `Mix.RemoteConverger.get()` (the registered Hex module, set via `register/1`), and `scm` is an SCM module already resolved by `Mix.SCM`/`Mix.Dep.Loader` upstream — neither is a raw atom taken from dependency metadata in this file, so this is not externally-influenced module dispatch. The network/fetch sinks themselves live behind those delegates in Hex and the SCM modules, not here.
- **`Mix.raise` / `Mix.shell().error` string interpolation (25, 38–41, 281–283).** The interpolated values (`app` atom, `inspect(dep.app)`, `find_cycle!` join) land only in a developer-facing error/warning string; there is no downstream interpreter (shell, query, regex, EEx), so no injection sink.
- **`:digraph` construction in `topological_sort/1` (15–45).** Creates protected ETS tables but deletes them in an `after` clause, so a raise mid-sort cannot leak them. Vertices/edges are bounded by the dependency count. `:digraph.add_edge/3` on a missing child vertex returns `{:error, _}` rather than raising, a benign correctness nuance with no security impact.
- **O(D²) traversal / topsort (`match_deps` `Enum.split_while`, `split_non_fulfilled_optional`, `Enum.find`-in-`Enum.map`).** Quadratic in the number of dependency nodes, but D is bounded by the transitive dependency-declaration count and this cost is dwarfed by the actual fetch/compile of those same deps. Inflating D to DoS-relevant size requires a malicious dependency whose `mix.exs` is evaluated anyway (arbitrary code execution by design, per the folder threat model), so there is no amplification over what the attacker already has.
- **`put_lock/2`, `req_mismatch/2`, `converge?/2`, `opts_equal?/2`.** Pure comparison/merge over struct fields; `~w(app env compile manager)a` is a compile-time literal list; version-requirement parsing happens in `Version`/`Mix.Dep.Loader`, not here.

## Risks / caveats
- The genuinely dangerous operations for this subsystem — converting dependency/app names from `mix.exs` or Hex registry metadata into atoms, network fetches, git-ref handling, lock-file deserialization, and archive/file writes — occur in sibling files (`Mix.Dep.Loader`, `Mix.RemoteConverger`/Hex, `Mix.SCM.*`, `Mix.Dep.Lock`). This file consumes their already-structured output, so those boundaries were treated as upstream and not re-audited here; they deserve their own review.
- I did not exercise the full `converge/4` path end-to-end against a crafted malicious dependency tree (that requires the loader and a remote converger); my conclusions about data reaching this file's sinks rest on static tracing of struct provenance plus the runtime confirmation of the bounded atom set, which is sufficient because every in-file sink is either bounded or a delegate to an upstream-resolved module.

