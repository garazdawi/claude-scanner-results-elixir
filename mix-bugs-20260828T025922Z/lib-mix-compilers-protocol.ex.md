# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/compilers/protocol.ex
<!-- sha256: 1bdf277f874b23d54af11a4e95bbff361ce5f6c116935c008ea3fe341de75afd -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:10:28Z_

This file is byte-for-byte identical to the shipped Elixir 1.21.0-dev `Mix.Compilers.Protocol`. I traced every public entry point (`umbrella/2`, `status/2`, `compile/4`, `clean/0`) and the supporting private logic. Let me record my analysis.

## Summary

`Mix.Compilers.Protocol` drives protocol consolidation for the Mix compiler (manifest read/write, diffing changed protocols/impls, and parallel consolidation); after tracing the control flow and the manifest/diff logic against the `Protocol` module's actual contracts, I found no genuine reachable bug.

## Bugfixes

No findings

## Risks / caveats

- The file is identical to the released Elixir 1.21.0-dev source (`diff` against `/opt/elixir/lib/mix/lib/mix/compilers/protocol.ex` reports no differences), so it is mature, well-exercised code; I looked for real defects regardless, but the base rate here is low.
- `Mix.Compilers.Protocol` is `@moduledoc false`. I treated its callable functions (`umbrella/2`, `compile/4`, `status/2`, `clean/0`) as the API surface driven by `Mix.Tasks.Compile.Protocols`/the compiler loop, rather than an external documented contract.
- I did not execute the module end-to-end (it needs a full project/build-lock/consolidation-path environment); findings would have been reasoned-plus-source-cross-checked rather than run-in-isolation. None reached the threshold of a finding.

### Ruled out

- **`read_manifest/1` `binary_to_term` on a corrupt manifest** — a truncated/garbage manifest or an old-version manifest (`[old_vsn | _]` failing the `[@manifest_vsn | metadata] =` match) raises inside the `try`, which is caught by the bare `rescue _`, cleaning consolidated output and returning the `{0, {%{}, %{}}}` default. The failure mode is handled, not a crash. Note `binary_to_term` runs on a trusted build artifact, not network input.
- **`diff_manifest/3` shape/threading** — I traced all four comprehensions: new-protocol timestamp bump (`new_timestamp > old_timestamp`), new impls, removed protocols (`remove_consolidated` returns the atom, fed to `Map.from_keys`), and removed impls guarded by `not is_map_key(removed_protocols, protocol)`. The accumulator `protocols` is threaded correctly through each `into:`; the mixed value types (`timestamp` vs `true`) are irrelevant because the function returns `Map.keys(protocols)`. No dropped/duplicated protocol.
- **`extract_protocols`/`extract_impls` receiving `{path, files}` tuples from `consolidation_paths/0`** — verified against `Protocol` source: its `@spec` and `list_dir/1` explicitly accept `{charlist, [charlist]}` pre-listed directories (`list_dir({path, files}) when is_list(path) and is_list(files)`), matching what `consolidation_paths/0` produces from `:code.get_path()`. No shape mismatch.
- **`status/2` `cond` fallthrough** — the three branches are exhaustive for the boolean/opts inputs (disabled → `:off`; missing consolidation path or mtime changed → `:force`; otherwise `:on`), so no `nil`/CondClauseError. `:consolidation_path` is a real config key (confirmed in `Mix.Project`).
- **`umbrella/2` status `case`** — `:off`, `:on when res == :noop`, and the catch-all `on_or_force` (matching `:on` with `res != :noop`, or `:force`) cover every value `status/2` can return; `on_or_force == :force` correctly distinguishes force from on.
- **`write_manifest` skipped on `:noop` compile** — leaves `config_mtime` unadvanced so a forced-but-empty run may re-force next time; this is a benign redundant no-op consolidation, not a behavioural defect, and matches upstream behaviour.
- **`consolidate/4` parallel tasks with `Task.await(:infinity)`** — a raising `consolidate_each` (e.g. `File.write!` failure) propagates through the linked task and aborts compilation, which is the intended Mix failure behaviour, not a leak or hang.

