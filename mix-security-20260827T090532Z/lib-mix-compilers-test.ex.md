# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/compilers/test.ex
<!-- sha256: f8db9b96a37e6fc18b8082772d87f33dad53a9d2bfadc7f13179ad8ea940dcd6 -->

_Analysed on 2026-08-27T09:27:29Z_

All three properties verified. Let me finalize the assessment. The sink is real, but the whole Mix codebase deliberately treats manifests as trusted local build state (every manifest uses plain `binary_to_term`; only the cross-process pubsub channel uses the restricted decoder), and the trust boundary is weak. I have what I need.

## Summary

`Mix.Compilers.Test` requires and runs a project's own test files; the only externally-derived bytes it decodes are its own build-state manifest, deserialized with `:erlang.binary_to_term/1` without `:safe`, which is exploitable only by an attacker who can already write inside `_build`.

## Security

#### `CSCAN-test-manifest-unsafe-binary-to-term`
- **Finding ID** — `CSCAN-test-manifest-unsafe-binary-to-term`
- **Severity** — Low
- **Confidence** — high
- **Location** — `elixir/lib/mix/lib/mix/compilers/test.ex:216`, `read_manifest/0` (payload written compressed at `:232`, `write_manifest/1`)
- **Class** — Deserialisation (CWE-502 / CWE-770 atom exhaustion / decompression amplification)
- **Impact** — The stale-test manifest at `<build_path>/lib/<app>/.mix/compile.test_stale` is decoded with plain `:erlang.binary_to_term/1`. An attacker who controls those bytes can (a) intern arbitrary atoms — the atom table is never GC'd, so a crafted manifest exhausts it and aborts the VM (CWE-770); and (b) exploit the fact that the manifest is written `[:compressed]`, so `binary_to_term` auto-inflates it: a ~190 KB file expands to 200 MB in-process. Both allocations happen *during* decode, before the `[@manifest_vsn | sources]` match, so the surrounding `try/rescue` does not prevent the resource spike (it only swallows the post-allocation `MatchError`/`ArgumentError`). The realistic delivery vectors are narrow — a project that commits/ships its `_build` (normally git-ignored), or a poisoned CI build-cache restored before a trusted `mix test`. Anyone who can write `_build` can usually also plant a `.beam` that Mix loads (direct code execution), so this is a defense-in-depth gap, not the weakest link — hence Low. Note this pattern is consistent across the whole Mix folder (every manifest reader uses plain `binary_to_term`; only `Mix.Sync.PubSub`, a genuine cross-process channel, uses the restricted `Mix.Utils.non_executable_binary_to_term`), i.e. the design intent is "manifests are trusted local state."
- **Evidence** — `verified`.
  - Atom interning + `:safe` rejection (fresh VM): `elixir -e '<<131,100,byte_size(name)::16,name::binary>>'` → `:safe -> rejected (ArgumentError)`; plain decode → `unsafe -> interned, atom_count delta 1`.
  - Decompression amplification: `payload = :erlang.term_to_binary([2 | :binary.copy(<<0>>,200_000_000)], [:compressed])` → `on-disk (compressed) size : 194428 bytes`; `[2 | inflated] = :erlang.binary_to_term(payload)` → `VM memory delta on decode : 200 MB from a 194428-byte file`.
- **Suggested fix** — There is no clean drop-in: `:safe` blocks the atom-exhaustion facet but, as Mix's own `non_executable_binary_to_term` docstring notes, would reject legitimate manifests whose module-name atoms aren't interned yet, and it does nothing for the compression-inflation facet. The robust fix is to bound the input before decoding — reject/rebuild the manifest if the on-disk file exceeds a sane size cap, and decode via a length-limited path (e.g. validate against `byte_size` and treat oversize/over-inflated payloads as "no manifest," matching the existing `rescue -> %{}` fallback) rather than trusting `[:compressed]` auto-inflation. Treat this consistently with the sibling manifest readers if changed at all.

### Ruled out
- **`Kernel.ParallelCompiler.require(test_files, …)` compiling/running code (lines 76, 89)** — `test_files` are the project's own matched test sources and helpers; running the developer's own test code is the documented purpose, not a boundary crossing.
- **`Code.compiler_options/1` global mutation (lines 32/37) and `Application.put_env(:ex_unit, …)` (lines 68/72)** — values come from the caller's `elixirc_opts`/ex_unit config (developer-controlled), and both are restored in `after`.
- **`File.read!/rm/mkdir_p!/write!` on `manifest()` (lines 216–233)** — the path is `Path.join(Mix.Project.manifest_path(), @stale_manifest)` where `@stale_manifest` is a module constant; no external path component, no traversal.
- **`each_module`/`each_file`/`get_external_resources` path handling (lines 294–328)** — operate on compiled test modules and their developer-declared `@external_resource` paths; the only downstream use is `Mix.Utils.last_modified` (a stat), not a write/exec sink.
- **`shuffle/3` seeding `:rand.seed(rand_algorithm, {seed,…})` (lines 334–336)** — `seed`/`rand_algorithm` are ex_unit config controlling test order, not a cryptographic use; not attacker-influenced.
- **`CE.read_manifest(elixir_manifest)` (line 244)** — the decode sink lives in `Mix.Compilers.Elixir`, another file; out of scope here. The module atoms/paths it yields are only compared or stat'd in this file, never applied or executed.

## Risks / caveats
- I could not drive the full `require_and_run/4` → `read_manifest/0` path end-to-end (it needs a live Mix project + `:ex_unit` and the function is private), so the manifest-delivery step is reasoned about, not executed; the deserialization *behaviors* themselves (atom interning, `:safe` rejection, decompression inflation, allocation-before-match) are all reproduced directly against the built OTP 29 / Elixir 1.21.0-dev toolchain.
- Severity rests on the trust boundary: the manifest is a locally-generated `_build` artifact. If your threat model includes untrusted `_build` restoration (shared/poisoned CI caches, committed build dirs, artifacts unpacked from an untrusted source), re-weight this upward — but in that model the co-located `.beam` files are a strictly stronger sink, so fixing this alone would not close the exposure.
- This is a folder-wide pattern (`app_loader`, `compilers/elixir`, `compilers/erlang`, `compilers/protocol`, `dep/elixir_scm`, `tasks/format`, `tasks/deps.partition` all decode manifests the same way). I reviewed only this file as instructed; a consistent hardening decision would need to span them.

