# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.unlock.ex
<!-- sha256: 4d073bbaf18cd49b95b5e190d198a7d97827848095d45f8ac621dfb782b7ad34 -->

_Analysed on 2026-08-27T10:54:11Z_

I now have the full picture. Let me verify my understanding of the data flow one final time by confirming that `apps` on line 76 is bound solely from `OptionParser`'s positional args (CLI), not from lock data.

The trace is clear from the file: `run(args)` → `OptionParser.parse(args, ...)` → positional `apps` → `do_run(opts, apps)` → line 76 maps CLI strings through `String.to_unsafe_atom`. Lock-file data (`Mix.Dep.Lock.read()`) only reaches `Map.drop`/`Map.keys`/`Atom.to_string`/`Enum.join`/`inspect`. Everything reaching a sink here originates from the operator's command line.

## Summary

`Mix.Tasks.Deps.Unlock` is a manually-invoked, destructive Mix task whose every input (positional dep names, `--filter`, the boolean switches) is operator-supplied command-line data, and the only externally-influenceable data it touches (lock-file contents) reaches nothing more dangerous than `Map.drop`/`inspect`/`Enum.join`; no security finding crosses a trust boundary in this file.

## Security

No findings.

### Ruled out

- **`String.to_unsafe_atom/1` on line 76 (unbounded atom creation, CWE-770).** This is a genuine `:erlang.binary_to_atom/2` sink (verified: `atoms from String.to_unsafe_atom on 3 positional args: 3`), and normally it would be an automatic flag. But its input is `apps`, the *positional command-line arguments* to `mix deps.unlock foo bar` (bound from `OptionParser.parse(args, …)` in `run/1`). There is no path by which a dependency, `mix.lock`, `mix.exs`, an archive, a git ref, or a network peer controls these args — `deps.unlock` is never invoked automatically by another task; it is a standalone, destructive, operator-typed command. Per this codebase's scope rules, operator-controlled input is a trusted boundary (an operator who can type the command line already controls the build machine), so exhausting the atom table this way is self-inflicted, not an attack. Defense-in-depth nit only: since the code immediately discards args not present in the lock (`apps -- unlocked`), `String.to_existing_atom/1` (treating "atom does not exist" as "not locked") would avoid interning typos/garbage entirely — but this is a hardening suggestion, not a vulnerability.
- **`--filter` ReDoS / pattern injection (line 66).** `Atom.to_string(&1) =~ filter` where `filter = opts[:filter]` is a `:string`. Verified that `=~` with a binary right operand performs a substring match (`String.contains?`), not regex compilation (`binary =~ binary … : true`), so there is no attacker-compiled regex and no catastrophic backtracking. The value is operator-supplied regardless.
- **Lock-file contents reaching a sink.** `Mix.Dep.Lock.read()` returns a map whose atom keys could, in principle, be influenced by a malicious `mix.lock` in an untrusted PR. Traced every use in this file — `Map.drop/2`, `Map.keys/1`, `Atom.to_string/1`, `Enum.join/2`, `Enum.sort/1`, `inspect/1`, `Enum.map_join/3` — all benign. The keys are *already atoms* (created upstream in `Mix.Dep.Lock`), so no new atom creation, no eval, no path/command/network sink is fed by lock data here.
- **`--check-unused` in CI on untrusted PRs.** The moduledoc advertises this for pre-commit/CI use, where `mix.exs`/`mix.lock` come from an untrusted contributor. The RCE risk there is inherent to Mix *evaluating* `mix.exs`/`mix.lock` (via `Mix.Project.get!/0` on line 31, `Mix.Dep.Converger.converge/1` on line 96, and `Mix.Dep.Lock.read/0`) — all in *other* files and by explicit Mix design ("running any Mix command on an untrusted project is RCE"). Nothing in *this* file adds or worsens that sink.

## Risks / caveats

- **Out-of-file trust anchors I did not audit.** This review is confined to `deps.unlock.ex`. It relies on `Mix.Dep.Lock.read/0`, `Mix.Dep.Lock.write/1`, `Mix.Dep.Converger.converge/1`, `Mix.Project.get!/0`, and `Mix.Project.with_deps_lock/1` behaving safely. In particular, `Mix.Dep.Lock.read/0` parses/evaluates the `mix.lock` file (Elixir source) — if that evaluation is not sandboxed, running any `deps.unlock` variant on an untrusted project is arbitrary code execution, but that belongs to those files' review, not this one.
- **OptionParser atom minting (minor, unresolved measurement).** In lenient mode, `OptionParser.parse` can intern an atom for an unknown `--some-flag`; I observed `:zzz_brand_new_flag_12345` appear in one parse result, though `:erlang.system_info(:atom_count)` reported a delta of 0 in this OTP build (likely a counter-granularity artifact — the direct `String.to_unsafe_atom` measurement showed the expected delta of 3). Either way the flags are operator-supplied CLI input, so this does not change the no-finding conclusion; I note it only because the exact atom-count instrumentation was not fully reconciled.
- All behavioral claims above (`=~` substring semantics, `String.to_unsafe_atom` atom creation) were **verified** by running Elixir built from the source under review; the trust-boundary reachability judgment (no non-operator path to `run/1` args) is **asserted** from static tracing of this file plus the fact that no Mix task auto-invokes `deps.unlock`.

