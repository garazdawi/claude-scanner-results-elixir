# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/compilers/elixir.ex
<!-- sha256: 24910c44b4d85e4cc21af93a415bbb663fafff97ba506d31e352d4de9a202685 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:08:56Z_

## Summary

`elixir.ex` is the Elixir incremental-compilation manifest engine for Mix — it is byte-for-byte identical to the compiler that the sandbox's `/opt/elixir` toolchain is built from (verified by `diff`, exit 0), and after enumerating the candidate defects and empirically driving the most suspicious path I found no genuine, reachable bug.

## Bugfixes

No findings.

## Risks / caveats

- **What I verified.** The file under review is identical to `/opt/elixir/lib/mix/lib/mix/compilers/elixir.ex`, which is the source the working `elixir`/`mix` in this sandbox was compiled from — so any defect that fires on a clean build would already break the toolchain. I also exercised the documented public surface directly: `read_manifest/1` (valid manifest → `{map, map}`; missing file → `{[], []}`; truncated/garbage file → `{[], []}` via the `rescue`), `diagnostics/1` (missing → `[]`), and `protocols_and_impls_from_paths/1` (missing path → `{%{}, %{}}`) all behaved to contract, including on corrupt input.
- **Main hypothesis, falsified by running.** I suspected `each_cycle/5` (line 1226–1230): the reduce seeds its accumulator with `stale_exports`, `Map.put`s the modules of every changed source into it, then destructures the result against the literal `{sources, %{}}` — which raises `MatchError` unless the accumulator ends empty. This branch is only reached on an *incremental* export-dependency cascade (not a clean build), so a self-compile of Elixir would never hit it. I built a two-module project with a real struct export dependency, changed the struct fields to force the cascade, and observed two compile cycles complete with exit 0 — the `changed != []` branch ran and the assertion held. The invariant (stale_exports fully drained and the changed sources' records already stripped of modules by the time the branch runs) is maintained upstream in the loop; the code is correct.
- **What I could not exhaustively verify.** `compile/7` is a long, stateful state machine coordinating `Kernel.ParallelCompiler` across `after_compile`/`each_cycle`/`each_file`/`each_module` callbacks and checkpoint/manifest versioning. I reasoned through the state-threading and accumulator paths and spot-checked the reachable branches, but I did not enumerate every combination of `force?`/`deps_changed?`/`reinfer?`/consolidation and manifest-migration inputs. My confidence that nothing is wrong rests substantially on the file being the unmodified, in-use upstream compiler rather than on independently proving each path; a subtle latent bug in a rarely-hit incremental combination cannot be fully excluded from static reading alone.

