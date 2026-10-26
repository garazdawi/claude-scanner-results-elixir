# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/failures_manifest.ex
<!-- sha256: 4436ac8ec8dd950597701bbab59c4611ca7446b5e0263e339f7f86fc3618a6f5 -->

_Analysed on 2026-08-27T05:25:22Z_

I have everything I need. Let me confirm the inventory is complete by checking there are no other sinks I've missed in the file (atom/eval/dispatch/path sinks beyond what I traced).

The file's only external-data entry is the manifest binary via `File.read` → `safe_binary_to_term`. The `File.*`/`Path.dirname` calls operate on the caller-configured `failures_manifest_path` (set by Mix, not attacker-derived), and the `File.regular?`/`function_exported?`/`Code.loaded?` calls in `find_deleted_tests` are read-only checks that don't load/dispatch code — they're downstream of the same decode. One finding stands.

## Summary

`ExUnit.FailuresManifest` serialises/deserialises the `mix test --failed` failures manifest to/from a file on disk; its security posture is sound except for one classic unsafe-deserialisation defect: it decodes the on-disk manifest with `:erlang.binary_to_term/1` **without** the `:safe` option, so a planted manifest file becomes an atom-table/resource-exhaustion (CWE-502/CWE-770) primitive reached on every `mix test`.

## Security

#### `CSCAN-failures_manifest-unsafe-binary-to-term`

- **Finding ID** — `CSCAN-failures_manifest-unsafe-binary-to-term`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `failures_manifest.ex:71`, `safe_binary_to_term/1` (reached from `read/1` at line 47 and `info/1` at line 57)
- **Issue** — `safe_binary_to_term/1` calls `:erlang.binary_to_term(binary)` with no `[:safe]` option on bytes read straight from the manifest file (`File.read(file)`). Despite the name, the wrapper only guards against malformed input (it `rescue`s `ArgumentError`); it does nothing to stop `binary_to_term` from **creating arbitrary atoms, funs, pids and refs** while decoding. Critically, the decode runs to completion (with all its side effects) *before* the `{@manifest_vsn, %{} = manifest}` pattern match, so the version/shape "validation" provides zero protection — a file whose version isn't `1` still injects every atom it contains. Reached by `ExUnit.RunnerStats` on `:suite_started` (`runner_stats.ex:76-78`) whenever `:failures_manifest_path` is configured, which Mix does by default for `mix test`.
- **Class** — Deserialisation (unsafe `binary_to_term`; CWE-502), compounding into Atom creation / Resource consumption (CWE-770).
- **Impact** — An attacker who can write the manifest file (a co-located user with write access to `_build`; a poisoned CI cache that restores `_build`/manifest artifacts across runs; a distributed repo/tarball that ships a crafted `_build`) causes the next `mix test` to decode hostile bytes. A ~1 MB manifest encoding ~1M unique atoms exhausts the atom table (default limit 1,048,576) and crashes the BEAM node running the suite — a denial of service on developer machines and CI runners. Because atoms are never garbage-collected, even sub-limit files leak permanently. Arbitrary funs/pids/refs are also materialised (though the manifest's consumers never invoke the funs, so this is resource exhaustion, not RCE). Secondary: on atom-table exhaustion `binary_to_term` raises `:system_limit`/`SystemLimitError`, which the narrow `rescue ArgumentError` does **not** catch, so it escapes as an uncaught crash rather than degrading to `new()`.
- **Evidence** — `verified`.
  - Atom injection via `read/1`, and validation-runs-after-decode:
    ```
    valid-manifest atom BEFORE read: absent
    read/1 returned: %{{:cscan_novel_9f3a, :cscan_test_9f3a} => "some_file.exs"}
    valid-manifest atom AFTER read:  EXISTS
    bad-version atom BEFORE read: absent
    read/1 (bad vsn) returned: %{}
    bad-version atom AFTER read:  EXISTS   <-- decoded despite version pattern-match failing
    ```
  - DoS magnitude (one `read/1` of a 50k-atom manifest) and fun materialisation:
    ```
    atom_count before=13748 after read of 50k-atom manifest=63779  delta=50031
    atom_limit=1048576
    value is a fun?: true
    invoking the decoded fun yields: :pwned
    ```
  - Fix behaviour, fresh VM, novel atom: current code creates it; `[:safe]` refuses and raises `ArgumentError` (already caught by the existing `rescue`), atom stays absent:
    ```
    no-safe:  before=absent  decode={1, %{{:cscan_novel_9f3a, ...} => "some_file.exs"}}  after=EXISTS
    safe:     before=absent  decode={:raised, ArgumentError}                              after=absent
    ```
- **Suggested fix** — Decode with `:erlang.binary_to_term(binary, [:safe])`. The manifest legitimately contains only atoms already present in the running suite (test module names and test-function names are compiled/loaded before the manifest is consulted), so `:safe` rejects nothing legitimate, and its `badarg` on hostile data is already handled by the existing `rescue ArgumentError -> :error` → graceful `new()`. Optionally broaden that rescue to also catch `SystemLimitError`/`ErlangError` so a pathological file degrades to `new()` instead of crashing the run.

### Ruled out

- `File.mkdir_p!` / `File.write!` / `File.read` / `Path.dirname` (lines 33–34, 40–41, 46, 56) — all operate on the caller-supplied `file` (the Mix/operator-configured `:failures_manifest_path`), not on any component derived from the manifest contents. No external path component, no traversal. Trusted caller argument.
- `File.regular?`, `Code.loaded?`, `function_exported?(mod, name, 1)` in `find_deleted_tests/3` (lines 89, 96) — on the read-then-write cycle these do run on `mod`/`name`/`file` values decoded from an attacker manifest, but `File.regular?` is a read-only existence check (its boolean only decides whether to drop an entry) and `function_exported?`/`Code.loaded?` do **not** load or dispatch code (they return false for unloaded modules). No code-execution or path-write primitive; the only real harm (atom creation for `mod`/`name`) is already the decode issue above.
- `find_deleted_tests/3` recursion (lines 80–105) — the "recurse on the same list to populate the file-existence cache" branch processes each element at most twice; it is linear in manifest size with no re-scan, so no algorithmic-complexity DoS originating in this file.
- `rescue ArgumentError` in `safe_binary_to_term/1` as dead code — not dead; it is reached and exercised (corrupted-binary path, covered by `failures_manifest_test.exs:174-176`). It is *insufficient*, not unreachable; captured within the main finding.

## Risks / caveats

- **Module is `@moduledoc false`** (internal to ExUnit). I treated its exported functions as in-scope because they are reached from a *documented* ExUnit option (`:failures_manifest_path`, `ex_unit.ex:351`) via `ExUnit.RunnerStats` on `:suite_started`, i.e. on a normal `mix test` run, and because the folder threat model explicitly designates `read/1`/`info/1` as the manifest-deserialisation boundary. If the review's scope is read strictly as "only functions with `@doc`", a triager could downgrade this, but the file-format-handling-from-an-entry-point rule squarely applies.
- **Exploitability is gated on filesystem write access** to the manifest (normally under `_build`). The severity assumes a plausible privilege crossing — CI cache poisoning that restores `_build`, a hostile repo/tarball that ships build artifacts, or a co-located writer — rather than a network attacker. An attacker who can *only* achieve this via already having code-exec on the box gains little; the Medium rating reflects the CI-cache / shipped-artifact vectors where the writer is not yet the test-runner. Adjust down to Low if your environment guarantees `_build` is never attacker-writable and never restored from an untrusted cache.
- I did not trace how Mix computes the default `:failures_manifest_path` (that logic lives in the `mix` tree, outside this folder); the finding does not depend on the exact path, only on the manifest being a non-integrity-protected on-disk file that `read/1`/`info/1` decode. Confirm the default location and whether any project commits or caches it to assess real-world reach.
- The embedded-fun result is a term-instantiation demonstration only; I confirmed the manifest's consumers (`RunnerStats`, `find_deleted_tests`) never call values as functions, so this is not a remote-code-execution path — the realistic impact ceiling is DoS/resource exhaustion.

