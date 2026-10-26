# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/elixir_scm.ex
<!-- sha256: 7868d1dcb8c1151f171c1cec0b2ce05c3e938018b3ba94678c4c84c036dd39f9 -->

_Analysed on 2026-08-27T09:33:23Z_

Reproduction complete and decisive. Let me compose the report.

## Summary

`Mix.Dep.ElixirSCM.read/1` deserializes the `_build/.../.mix/compile.elixir_scm` manifest with `:erlang.binary_to_term/1` **without `:safe`**, so a tampered manifest interns unbounded novel atoms (verified: a 340 KB file interned 20,000 atoms; exhaustion aborts the VM uncatchably) on a path that runs before nearly every `mix` command.

## Security

#### `CSCAN-elixir_scm-unsafe-binary-to-term`
- **Finding ID** — `CSCAN-elixir_scm-unsafe-binary-to-term`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `elixir/lib/mix/lib/mix/dep/elixir_scm.ex:31`, `Mix.Dep.ElixirSCM.read/1` (inside the `try`)
- **Class** — Deserialisation (CWE-502), yielding atom-table exhaustion (CWE-770) / resource-consumption DoS
- **Impact** — The manifest at `_build/<env>/lib/<app>/.mix/compile.elixir_scm` (and each dependency's `<build>/.mix/compile.elixir_scm`) is read on essentially every `mix` invocation: `read/1` is called from `mix/tasks/loadpaths.ex:61` (runs ahead of `compile`/`run`/`test`/…) and per-dependency from `mix/dep/loader.ex:431`. Because the bytes are fed to `binary_to_term/1` with no `:safe`, an actor who can write that file — realistically via **CI/build-cache poisoning** (`_build` is routinely cached keyed on the lockfile hash), a shipped/committed `_build`, or a co-tenant on a shared build host — makes decoding intern every atom named in the file. Pushing the atom table to its limit aborts the whole BEAM node with `no more index entries in atom_tab`, an **uncatchable** crash the surrounding `try/rescue` cannot stop, breaking the build pipeline persistently and cheaply (it survives even a full recompile, since the read happens first). Decoded terms are only `!=`-compared by the callers, so this is a DoS, not RCE. Severity would rise to Medium in shared-cache/CI environments where `_build` crosses a trust boundary.
- **Evidence** — `verified`.
  - Novel atoms are interned by the unsafe decode and rejected by `:safe`:
    ```
    before decode, atom is: absent
    unsafe binary_to_term returned: :cscan_novel_1573_zzz  (is_atom=true)
    after unsafe decode, atom is: exists
    safe binary_to_term([:safe]) on fresh novel atom: {:raised, "... invalid or unsafe external representation of a term ..."}
    ```
  - Driving the real `Mix.Dep.ElixirSCM.read(dir)` with a 340 KB list-of-atoms manifest — the `try/rescue` fires (returns the fallback) yet the atoms are already interned:
    ```
    read/1 returned        : {:ok, {"1.0.0", ~c"17"}, nil, nil, []}
    probe atom absent before read: true
    probe atom exists after read : true
    atom_count delta       : 20000  (encoded 20000 novel atoms)
    ```
  - Terminal consequence via the same `read/1` path (atom limit forced low for a cheap deterministic hit); the rescue never regains control:
    ```
    no more index entries in atom_tab (max=65536)
    Crash dump is being written to: erl_crash.dump...done
    === elixir exit status: 1 ===   ("SURVIVED" was never printed)
    ```
- **Suggested fix** — Decode with the safe option: `:erlang.binary_to_term(contents, [:safe])`. This is behavior-preserving — the existing `rescue` already converts a bad decode into the `{:ok, {"1.0.0", ~c"17"}, nil, nil, []}` "stale manifest → recompile" fallback, and `:safe` raises `badarg` on a novel atom, so the bomb now takes that same graceful path with **zero** atom growth (verified: `with [:safe] -> returned: {:ok, {"1.0.0", ~c"17"}, nil, nil, []}` / `atom_count delta with [:safe]: 0`). Optionally pair it with a byte-size cap on `contents` before decoding, since `:safe` bounds term *type* but not raw allocation size. Note the one behavioral tradeoff to weigh: `:safe` would reject a *legitimate* manifest that names an atom (e.g. a custom `Mix.SCM` module) not yet interned in the reading VM, degrading to a forced recompile rather than a crash — the conservative, safe direction, and in practice such atoms (SCM module, dep app names) are already loaded when `read/1` runs.

### Ruled out
- `manifest/1` (`Path.join(manifest_path, @manifest)`) — `@manifest` is the constant `"compile.elixir_scm"` and `manifest_path` is operator/Mix-derived build config, not attacker input; no traversal component crosses a boundary.
- `update/4` — `term_to_binary` is *serialization* (no term instantiation from untrusted bytes), and `File.mkdir_p!`/`File.write!` target the operator-controlled build path with operator-controlled data (`scm`/`lock`/`deps` from project config).
- RCE / module-dispatch via the decoded term — the decoded `scm`/`lock`/`deps` are only compared with `!=` in `loadpaths.ex:62` and `loader.ex:432`, never `apply`-ed or used as a call target, so `:safe`'s fun/pid/port protection here is defense-in-depth; the operative impact is the atom/resource DoS above.

## Risks / caveats
- The core dispute is the trust boundary, not the mechanics (which are fully reproduced end-to-end against the real `read/1`, including the uncatchable VM abort). `_build` is normally within the developer's own trust domain; the finding matters only where that directory can be influenced by another party (CI cache poisoning, shipped/committed `_build`, shared build hosts) — hence Low severity / medium confidence. Anyone with arbitrary write into `_build` can usually also plant a malicious `.beam` (a stronger RCE primitive), which is why this is scored as an incremental, cheap, recompile-surviving DoS rather than a novel escalation.
- I did not exercise a live CI-cache-poisoning scenario end-to-end (out of sandbox scope); the cache-poisoning vector is asserted as a known supply-chain class, while the decode/crash behavior it would trigger is verified.
- Default atom-table limit is ~1,048,576; I demonstrated exhaustion with a reduced `+t 65536` limit for speed. At the default limit the same crash needs a proportionally larger (~tens of MB) manifest in one file, which is feasible for a targeted DoS but I did not generate the full-size file.

