# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/compilers/elixir.ex
<!-- sha256: 24910c44b4d85e4cc21af93a415bbb663fafff97ba506d31e352d4de9a202685 -->

_Analysed on 2026-08-27T09:22:02Z_

## Summary

`Mix.Compilers.Elixir` deserializes its on-disk manifest/checkpoint files with unsafe `:erlang.binary_to_term/1` and then uses the module names it reads back as filesystem paths, so an attacker who can influence a project's `_build` artifacts (a committed/shipped `_build`, a restored CI build cache, or a poisoned path-dependency build dir) can crash the build VM or delete arbitrary `.beam` files during an ordinary `mix compile`.

## Security

#### `CSCAN-elixir-manifest-unsafe-binary-to-term`
- **Finding ID** — `CSCAN-elixir-manifest-unsafe-binary-to-term`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `lib/mix/lib/mix/compilers/elixir.ex:356` (`read_manifest/1`); same defect at `:932` (`parse_manifest/2`) and `:1020` (`parse_checkpoint/2`)
- **Class** — Deserialisation (CWE-502), Atom creation (CWE-770), Resource consumption / decompression bomb (CWE-409)
- **Impact** — The manifest (`_build/<env>/lib/<app>/.mix/compile.elixir`) and its `.update.cp`/`.verify.cp` checkpoints are read with `File.read!() |> :erlang.binary_to_term()` — no `[:safe]`. The decode runs *before* any version/shape check, so any bytes are fully deserialized as a side effect even when the result is discarded. An attacker who controls those files (committed `_build` in a cloned repo, a restored CI cache keyed on attacker input, or a local path-dependency's build dir read via `stale_local_deps`/`protocols_and_impls_from_paths`) gets: (a) creation of arbitrary permanent atoms → atom-table exhaustion → VM abort, and (b) because the manifest is written with `[:compressed]`, a decompression/allocation bomb — a ~300 KiB file forces ~300 MiB allocation (~1030×), so a few-KB file OOM-kills the build. Reachable via the documented public `read_manifest/1`, `diagnostics/1`, `protocols_and_impls_from_paths/1`, and internally during `compile/7`/`clean/2`.
- **Evidence** — verified.\
Atom injection through the public API (fresh atom, never created in-VM, appears after the call even though the versioned tuple doesn't match):
```
$ elixir /tmp/repro_atom3.exs
atom BEFORE (via public read_manifest): :does_not_exist
read_manifest returns: {[], []}
atom AFTER  (created as side effect => unsafe decode): :exists
```
Allocation amplification via the same public path:
```
$ elixir /tmp/repro_bomb.exs
on-disk poisoned file: 305777 bytes (298.6 KiB)
peak allocation forced by one read_manifest call: ~300.4 MiB
amplification factor: ~1030x
```
- **Suggested fix** — Decode with `:erlang.binary_to_term(bin, [:safe])` and treat a decode failure like a corrupt manifest (`@default_manifest` / `{[], []}`); additionally bound the accepted term/decompressed size before use. `:safe` also neutralizes the traversal atoms in the next finding, since they would not decode.

#### `CSCAN-elixir-manifest-module-path-traversal`
- **Finding ID** — `CSCAN-elixir-manifest-module-path-traversal`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `lib/mix/lib/mix/compilers/elixir.ex:62` (verify-checkpoint delete loop in `compile/7`) and `beam_path/2` at `:1014-1016` as used by `clean/2` (`:347`), `remove_and_purge` (`:376`, `:639`, `:970`)
- **Class** — File operations and path handling / path traversal (CWE-22)
- **Impact** — Module names are taken from the unauthenticated manifest/checkpoint as atoms and used verbatim to build filesystem paths: `Path.join(dest, Atom.to_string(module) <> ".beam")` followed by `File.rm`. There is no check that the atom is a real module name; `:"../../../../x"` escapes `dest`. A poisoned `.verify.cp` (or a v37 manifest for `clean/2`) yields arbitrary file deletion, bounded to paths ending in `.beam` — enough to delete the compiled artifacts of *other* apps or the OTP libraries installed on the build/CI machine, breaking unrelated installations. Deletion fires on a plain `mix compile`, not only `mix clean`.
- **Evidence** — verified. Poisoned `_build/.../compile.elixir.verify.cp` = `{5, [:"../../../../../../../../../../tmp/cscan_cp_victim"]}`, then an ordinary compile:
```
$ cd /tmp/proj && mix compile   # victim lives OUTSIDE the project
victim BEFORE mix compile: -rw-r--r-- ... /tmp/cscan_cp_victim.beam
victim AFTER  mix compile: ls: cannot access '/tmp/cscan_cp_victim.beam': No such file or directory
```
`clean/2` shows the same via `beam_path` (deleted `/tmp/cscan_victim.beam` outside `compile_path`). A NUL byte in the atom does **not** truncate the appended `.beam`, so deletion stays `.beam`-suffixed.
- **Suggested fix** — Before pathing a module name, reject any whose `Atom.to_string/1` contains a path separator or `..` (e.g. require `Path.basename(str) == str`), and/or decode manifests with `[:safe]` so traversal atoms never materialize.

#### `CSCAN-elixir-purge-path-unsafe-atom`
- **Finding ID** — `CSCAN-elixir-purge-path-unsafe-atom`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `lib/mix/lib/mix/compilers/elixir.ex:781` (`purge_modules_in_path/1`)
- **Class** — Atom creation (CWE-770)
- **Impact** — Each filename found in `opts[:purge_consolidation_path_if_stale]` (the consolidated-protocols build dir) is converted to an atom via `String.to_unsafe_atom/1`. Under the same attacker-influenced-`_build` boundary, populating that directory with many uniquely named `*.beam` files exhausts the atom table on the next stale-consolidation purge. Normal builds place only a handful of files there, so reachability is conditional on a poisoned build dir.
- **Evidence** — asserted (sink is `String.to_unsafe_atom`; reachability gated by an internal opt and the consolidation-stale branch).
- **Suggested fix** — Use `String.to_existing_atom/1` and skip names that do not resolve (a module whose beam is being purged should already exist as an atom), or bound the number of names processed.

### Ruled out
- `module.__mix_recompile__?()` / `Code.ensure_all_loaded/1` (`:424-437`) executing and loading modules named in the manifest — this is by-design compile-time execution of the project's own build graph; the callback invoked is the Mix-specific `__mix_recompile__?/0`. (A poisoned manifest can force-loading of arbitrary *already-compiled* modules, running their `@on_load`; see caveats — the lever is the deserialization finding, not a separate sink.)
- `each_file` `File.read!` (`:1312`) and `@external_resource` reads in `process_external_resources`/`digest_changed?` — the former reads the project's own source being compiled; the latter reads dependency-declared paths only to compute a digest that is never exfiltrated (documented feature).
- `each_long_compilation`/`debug_recompile` string interpolation (`:1102`, `:897`) — interpolated values are project source paths / the app atom; the target is `Mix.shell()` output, no injection sink.
- `Task.async_stream(..., timeout: :infinity)` (`:428`) — fan-out is bounded by the number of manifest modules, not an unbounded attacker input.

## Risks / caveats
- **Trust boundary is the crux, not the mechanism.** All findings assume the `_build` manifest/checkpoint (or a path-dependency's build dir) is attacker-influenced. Under the folder's strict threat model the operator's own `_build` is trusted; the realistic in-scope deliveries are a *committed/shipped* `_build` in a cloned untrusted repo (opened + compiled, possibly by an editor/LSP automatically), a *restored CI build cache*, and umbrella/path-dep manifests read by `stale_local_deps`/`protocols_and_impls_from_paths`. Severity is calibrated to that; a reviewer treating `_build` as fully trusted would downgrade these to defense-in-depth (Low).
- I verified `read_manifest/1` and `clean/2` directly, and the checkpoint-delete traversal end-to-end through `mix compile`. I did **not** separately drive the `parse_manifest`/`purge_old_manifest` old-format branches (`:944`, `:948`, `:970`) or `stale_local_deps` (`:672`) end-to-end, but they call the identical `binary_to_term`/`beam_path` primitives already shown vulnerable.
- A poisoned manifest can also cause `Code.ensure_all_loaded/1` to load arbitrary already-present modules (triggering any `@on_load`); I did not build a module with a malicious `@on_load` to demonstrate code execution, so I list it only as a caveat rather than a finding.
- `:safe` bounds atom/term type but does **not** stop the decompression bomb; a size cap on the decompressed manifest is needed for the DoS specifically.

