# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/runner_stats.ex
<!-- sha256: dc2856415567c0ce6296a5024d8a3102fbb6b1b48b50ae80bc38f610a2c34409 -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/runner_stats.ex
<!-- sha256: dc2856415567c0ce6296a5024d8a3102fbb6b1b48b50ae80bc38f610a2c34409 -->

_Re-reviewed on 2026-08-27_

## Summary

`ExUnit.RunnerStats` is an internal (`@moduledoc false`) `GenServer` event-handler that tallies test outcomes and drives the failures-manifest read/write via `ExUnit.FailuresManifest`. Its own public surface (`stats/1`, `get_failure_counter/1`, `increment_failure_counter/2`) is trivial and well-guarded, and it contains **no** code-execution, atom-creation, path-manipulation, or deserialization sink of its own. The one security-relevant sink reachable through it (unsafe term deserialization of a planted manifest) lives in `failures_manifest.ex`, not in this file.

This re-review resolves the two first-pass hedges by reading the callgraph neighbours `failures_manifest.ex` and `ex_unit.ex`:

- **Hedge 1 (manifest-path origin "not exhaustively traced"):** resolved. `failures_manifest_path` is a documented `ExUnit.configure/1`/`ExUnit.start/1` option (`ex_unit.ex:123` type spec, `ex_unit.ex:351-352` doc), consumed here purely as `opts[:failures_manifest_path]` (`runner_stats.ex:43`). Within this folder there is no path by which a network peer or dependency value reaches it — it is operator/CI config. Trusted origin **confirmed**, not assumed.
- **Hedge 2 (`test.tags.file` "treated as trusted, not analyzed"):** resolved. The only consumer of `test.tags.file` reachable from this module is `FailuresManifest.put_test/2` (`failures_manifest.ex:26`), which merely `Map.put`s the value as a manifest map-value; it is later re-serialized via `term_to_binary` in `write!/2` (`failures_manifest.ex:32`). The struct is built by the runner from developer-authored test source (see standing context / prior `exunit-runner-review`). It is stored data, never used as an atom key, path-execution target, or format string in this call chain. Trusted-and-inert **confirmed**.

## Security

No findings.

The file was traced sink-by-sink against the Phase 1 inventory. Every candidate either has its vulnerable code in another file, is driven only by trusted/operator inputs, or is reachable only by bypassing the documented API. Details under *Ruled out*.

### Ruled out

- **Unsafe `binary_to_term` on a planted failures manifest (CWE-502 / CWE-770) — sink is in another file.**
  `handle_cast({:suite_started, …})` (`runner_stats.ex:76-80`) calls `FailuresManifest.read(file)`, and this path is **live**: a planted manifest at the configured `failures_manifest_path`, decoded through this handler in a fresh VM, created 2030 novel atoms (GenServer stayed alive). The actual defect — `:erlang.binary_to_term(binary)` called **without `[:safe]`**, rescuing only `ArgumentError` — is `ExUnit.FailuresManifest.safe_binary_to_term/1` (`failures_manifest.ex:70-74`, re-read this pass to confirm the exact code). `runner_stats.ex` only stores the operator-configured path and forwards it; the fix belongs in `failures_manifest.ex`. Same for the `FailuresManifest.write!` sink at `runner_stats.ex:82-86` (writer is `term_to_binary`, `failures_manifest.ex:32`, benign). Carried as an actionable cross-file pointer below, not a finding against this file.
  Axes examined: sink location, liveness of the read path, atom-growth impact | Not examined: none.
- **`handle_call({:increment_failure_counter, increment}, …)` performs unguarded `failure_counter + increment` (`runner_stats.ex:61-64`).** A non-integer/negative `increment` would crash or corrupt the counter, but this clause is reachable only by sending a crafted message directly to the internal GenServer — the documented public wrapper `increment_failure_counter/2` enforces `is_integer(increment) and increment >= 1` (`runner_stats.ex:28-32`; verified previously that `increment(0)`/`increment(-5)` raise `FunctionClauseError`, fail-closed). Bypassing the wrapper is out of scope; the actor is already in-VM.
  Axes examined: guard on public wrapper, direct-cast bypass reachability | Not examined: none.
- **Counter drift / negative `passed` in `handle_cast({:module_finished, …})` (`runner_stats.ex:88-105`).** The `passed - 1` / `failures + 1` reclassification can produce inaccurate or negative tallies, but the inputs are internal `%ExUnit.TestModule{}` structs from the trusted runner; correctness concern, not a security boundary crossing.
  Axes examined: input provenance (runner-built struct), impact class (tally accuracy) | Not examined: none.
- **`is_binary(file)` guards on `:suite_started`/`:suite_finished` (`runner_stats.ex:76,82`).** Not dead — reachable when `failures_manifest_path` is configured (confirmed the option exists and is stored at `runner_stats.ex:43`), and correctly gates out the unconfigured (`nil`) case so no manifest I/O is attempted; verified previously the GenServer survives a `:suite_started` cast both with and without a valid path.
  Axes examined: reachability with/without config, nil-gating | Not examined: none.
- **Catch-all `handle_cast(_, state)` (`runner_stats.ex:107-109`) and `increment_status_counter(state, _)` (`runner_stats.ex:127`).** Defensive fall-throughs; verified previously that unknown casts (`{:totally_unknown_event, …}`, a bare atom) are swallowed without crashing.
  Axes examined: unknown-cast tolerance, unknown-tag tolerance | Not examined: none.
- **`pids: []` state field (`runner_stats.ex:46`).** Initialized and never read; dead state, no security relevance.
  Axes examined: read/write reachability | Not examined: none.

## Risks / caveats

- **Actionable cross-file pointer (not this file):** the term-deserialization sink reachable from this module's `:suite_started` handler is `ExUnit.FailuresManifest.safe_binary_to_term/1` in `lib/ex_unit/lib/ex_unit/failures_manifest.ex:70-74`, which calls `:erlang.binary_to_term/1` **without `[:safe]`** and rescues only `ArgumentError`. Verified that a planted manifest creates unbounded novel atoms (CWE-770) through this exact path. The decode occurs *before* the `{@manifest_vsn, %{} = manifest}` pattern match (`failures_manifest.ex:47`), so the pattern is not a guard — `:safe` is. `info/1` (`failures_manifest.ex:55-57`) shares the same weakness. Trust rests on the `.mix`/`_build` manifest directory not being attacker-writable. A follow-up should review/fix `failures_manifest.ex`.
- The two per-test-event caveats from the first pass are now resolved (see Summary): `failures_manifest_path` and `test.tags.file` are both confirmed trusted-origin inputs within this folder, and `test.tags.file` is stored inertly rather than used as an execution/atom/format sink in this call chain.

## Did I verify this?

- **Read this pass:** `runner_stats.ex` (full), `failures_manifest.ex` (full — confirmed the `binary_to_term` sink, the `put_test` handling of `test.tags.file`, and the `write!` serializer), `ex_unit.ex:345-359` (confirmed `:failures_manifest_path` is a documented config option), and a repo-wide grep for `failures_manifest_path` (only consumers: the two neighbours plus tests).
- **Executed (carried from prior pass, still valid — file sha unchanged `dc285641…`):** planted-manifest decode through `:suite_started` in a fresh VM → 2030 novel atoms, GenServer survived; public-wrapper guard rejects `0`/`-5` with `FunctionClauseError`; unknown casts swallowed; `:suite_started` survives with and without a configured path.
- **Confidence: high** that this file contains no in-file security finding. The single genuine sink is cross-file (`failures_manifest.ex`) and is flagged there.

## Dropped in re-review

None. No prior finding is dropped (the first pass reported no findings). Both first-pass hedges were upgraded to verified trust claims by reading `failures_manifest.ex` and `ex_unit.ex`; neither rose to a low-confidence finding because the reachable-from-this-file behaviour on those inputs is inert storage/forwarding, with the only exploitable decode located in a different file.
