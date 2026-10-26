# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.erlang.ex
<!-- sha256: 1b25fbe7e940f23492b8421057c388dbcb90571b6e18955a3587d5c8b8163aba -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.erlang.ex
<!-- sha256: 1b25fbe7e940f23492b8421057c388dbcb90571b6e18955a3587d5c8b8163aba -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`compile.erlang.ex` is the Mix task that compiles a project's `.erl` sources; it scans each file with `:epp` for dependencies/includes, topologically sorts by parse-transform/behaviour edges, and drives `Mix.Compilers.Erlang.compile_entries/6`. The compilation logic is sound. The one genuine defect remains the dead self-file guard in include scanning (benign impact). The first-pass hedge about the `parallel`/`compile_entries` interaction being "out of scope" is now resolved by reading `compilers/erlang.ex`: the interaction is verified consistent and correct — no bug.

## Bugfixes

#### `CSCAN-compile.erlang-self-include-guard-dead`
- **Finding ID:** `CSCAN-compile.erlang-self-include-guard-dead`
- **Severity:** Low
- **Confidence:** high
- **Location:** `compile.erlang.ex:175`, `do_form/3` (`:file` attribute clause), reached from `scan_source/4:165`
- **Class:** Binary vs charlist vs string (guard comparison across incompatible types)
- **Impact:** The guard `when file != include_file` is meant to stop the source file's own `-file` attribute (re-emitted by `:epp` when parsing returns from an `-include`) from being recorded as one of its own includes. `file` is the Elixir binary path passed into `scan_source/4`, while `include_file` comes out of `:epp` forms as a charlist, so `file != include_file` is *always* true and the guard never fires. As a result a source file that includes any header records itself in `erl.includes`. In practice this is harmless: `maybe_stale/3` checks `[erl.file | erl.includes]`, so the source file is already in the staleness set — the duplicate is a redundant `File.stat` with no effect on the stale/recompile decision. The finding is the dead guard itself; the intended de-duplication silently does nothing.
- **Evidence:** verified.
  - Comparison is always true: `elixir -e 'IO.inspect("src/foo.erl" != ~c"src/foo.erl")'` → `true`.
  - Real end-to-end reproduction with a source `foo.erl` that `-include("defs.hrl")`, driving `:epp.parse_file/3` then folding `do_form` exactly as the file does:
    ```
    === file attributes in forms ===
    ~c"src/foo.erl"
    ~c"include/defs.hrl"
    ~c"src/foo.erl"
    resulting includes: [~c"src/foo.erl", ~c"include/defs.hrl"]
    ```
    The intended result is `[~c"include/defs.hrl"]`; the source file leaks into its own includes.
- **Suggested fix:** Compare like-typed values — e.g. gate on the charlist form (`Erlang.to_erl_file(file) != include_file`), or drop the source path from `includes` in `maybe_stale/3`. Purely a cleanup unless `includes` is ever consumed for something order- or identity-sensitive.

## Risks / caveats

- **Reproduction scope (unchanged, not a claim about behaviour):** I could not stand up a full `Mix.Tasks.Compile.Erlang.run/1` inside a real project harness here. The include-scan defect was reproduced by driving `:epp.parse_file/3` + `do_form` with the exact code path, and the comparison/typing facts were run directly. This is a note on my test setup, not an unverified assumption about the code.
- **`Path.relative_to(charlist, binary)` at line 85 — verified correct.** `compile_path` is a charlist from `to_erl_file`; `Path.relative_to` returns a proper binary and the downstream `Path.join(compile_path, "#{module}.beam")` at line 213 works. Confirmed in first pass by execution; unchanged.

## Verified interactions (previously hedged as "out of scope")

- **Upgraded from hedged assumption in first pass** — the first pass wrote that `compile_entries/6`, the `:parallel` split, and manifest handling "live in `Mix.Compilers.Erlang` (out of scope for this file) … I did not independently audit it." I read `compilers/erlang.ex` in full and verified the specific interaction this file relies on. Result: **consistent, no finding.**

- **`parallel` set is type-consistent with the split in `compile_entries`.** In `compile.erlang.ex:216-219`, `parallel` is a list of the `file` binaries whose module has `:digraph.in_neighbours == []`; `opts[:parallel] = MapSet.new(parallel)` (line 89). In `compilers/erlang.ex:142-147`, the consumer does `Enum.split_with(stale, fn {source, _target} -> source in parallel end)`, where `stale = for {:stale, src, dest} <- mappings, do: {src, dest}` (line 108) and `src` is exactly the same `file` binary threaded through `sorted` (`compile.erlang.ex:213`, `{status, file, beam}`). Same value on both sides of `in` — the membership test cannot silently miss. Verified by reading, not assumed.
- **The parallel-vs-serial partition is semantically correct.** Edges are `module -> dep` (`compile.erlang.ex:203-207`, dep = behaviour/parse-transform). `in_neighbours(module) == []` selects modules that *nothing depends on*; any module that is depended upon has a non-empty in-neighbour set and therefore lands in `serial`. Consequence: (a) no two `parallel` modules can have a dependency edge between them (if A→D existed, D would have in-neighbour A and be serial), so the parallel set is mutually independent and safe for `Task.async_stream` at `erlang.ex:151-157`; (b) serial deps are compiled first (`erlang.ex:149`) before the parallel batch, so parse-transforms/behaviours are available. Walked A→B→C by hand and confirmed serial order is deps-first via the topsort (`compile.erlang.ex:209-214`, prepend-reduce reverses `:digraph_utils.topsort`'s u-before-v ordering into dep-before-dependent). No ordering bug.
- **Manifest handling is not affected by anything computed in this file.** `read_manifest`/`write_manifest` (`erlang.ex:265-280`) operate on `{dest, warnings}` tuples derived from the callback results, independent of the `parallel` set. The `dest_ext == :beam` modules-diff path (`erlang.ex:168-179`) consumes `stale` dests only. Nothing this file passes can put the manifest into a bad shape. Verified by reading.

### Ruled out
- **`scan_sources/5:148` `Enum.flat_map` lacks an `{:exit, _}` clause** — `Task.async_stream` would surface `{:exit, reason}` (unmatched by the two `{:ok, ...}` clauses → `CaseClauseError`) only if a `scan_source` task crashed. Verified `:epp.parse_file/3` returns `{:ok, forms}` (error nodes embedded, handled by `do_form`'s catch-all at `:191`) or `{:error, _}` (handled at `:168`) for syntax-error and missing-`-include` inputs, and neither `do_form` (only `File.regular?`) nor `maybe_stale` (`Mix.Utils.stale?`, which tolerates `File.stat` errors) raises on those inputs. Confidence: medium — the "no `scan_source` ever crashes" claim rests on the observed `:epp` return contract plus the non-raising helpers, not on an exhaustive enumeration of every filesystem race; a dismissal, and a later pass is free to re-open the crash path.
- **`:epp.parse_file/3` 3-tuple `{:ok, forms, extra}` return unhandled** — verified arity-3 `parse_file` returns a 2-tuple even for a file with a `%% coding: utf-8` directive; the 3-tuple variant belongs to `parse_file/2`. Both returned shapes (`:164`, `:168`) are matched. Confidence: high.
- **`tl(forms)` on empty forms (`:165`)** — `:epp` always emits at least an `eof` form, so `forms` is never empty on the `{:ok, _}` branch. Confidence: high.
- **`topsort_and_parallelize/2` cycle path calling `Mix.Utils.find_cycle!`** — confirmed the function exists (`utils.ex:1127`); the raise path (`:223-227`) is intact. `:digraph.add_edge` on a missing dep vertex returns `{:error, {:bad_vertex, _}}` (ignored by design at `:206`, per the comment), not a raise. Confidence: high.

## Did I verify this?

- **Self-include dead guard (`:175`):** Yes — executed. Type-mismatch comparison run directly; full `:epp.parse_file/3` + `do_form` fold reproduced the leaked self-include.
- **`Path.relative_to` at `:85`:** Yes — executed in first pass; returns a binary, downstream `Path.join` works.
- **`parallel`/`compile_entries` interaction (was "out of scope"):** Yes — read `compilers/erlang.ex:107-190` in full. Verified value-type consistency of the `parallel` MapSet against the `source in parallel` split, the semantic correctness of the no-in-neighbours partition (mutual independence + deps-first serial order), and that manifest handling is unaffected by this file's inputs. Verification is by reading the consumer's code, whose behaviour is directly determined by that code (not by an untried input axis), plus a hand-walked A→B→C ordering trace.
- **Ruled-out crash path (`flat_map`):** Partially — the `:epp` return contract was exercised; the "no helper ever raises under any filesystem state" generalization was reasoned, not exhaustively executed, hence the ruling is held at medium confidence rather than promoted to a claim of impossibility.

### Dropped in re-review
None. All first-pass findings and ruled-out entries are preserved. The only change is that the first-pass "out of scope / not audited" caveat on `Mix.Compilers.Erlang` has been discharged by reading the module and confirming the interaction is consistent — recorded above under "Verified interactions" rather than dropped.
