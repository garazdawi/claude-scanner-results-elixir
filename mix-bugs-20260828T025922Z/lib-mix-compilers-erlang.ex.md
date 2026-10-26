# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/compilers/erlang.ex
<!-- sha256: 7bccaa85365d9857f9a9f551d46e950209f4efbdcb529de1999b235de101b3da -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:11:18Z_

Based on my review of the file, here is my report.

## Summary

`Mix.Compilers.Erlang` orchestrates staleness detection, callback-driven compilation, manifest bookkeeping, and diagnostic formatting for `.erl`/`.yrl`/`.xrl`-style compilers; the code is largely sound, with one verified deviation from its documented contract around the `:preload` option.

## Bugfixes

#### `CSCAN-erlang-preload-runs-when-not-stale`
- **Finding ID:** `CSCAN-erlang-preload-runs-when-not-stale`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/mix/lib/mix/compilers/erlang.ex:93` — `compile/6`
- **Class:** Truthiness/contract confusion (`entries != []` used as a proxy for "something is stale")
- **Impact:** The `:preload` callback is documented as "any code that must be preloaded **if any pending entry needs to be compiled**." In practice it runs whenever any source file exists in the mappings, even when every file is up to date and nothing will be compiled. Real callers (`compile.yecc`, `compile.leex`) use `preload` to emit a configuration warning and to start `:parsetools`; both fire spuriously on a pure no-op build. A caller supplying a preload with heavier side effects (the type is `(-> term())`) has it invoked when the contract says it should not be.
- **Evidence:** `verified`. `entries` (line 88) collects *both* `{:stale, ...}` and `{:ok, ...}` tuples from `extract_entries`, so `entries != []` is true whenever any source file is present regardless of staleness. Driving `compile/6` with one up-to-date mapping:

  ```
  $ elixir preload_test.exs
  PRELOAD CALLED
  result: {:noop, []}
  preload_count: 1
  ```

  The compile callback was never invoked (result `:noop`), yet `preload` ran once. So preload fired despite nothing being pending.
- **Suggested fix:** Gate preload on the presence of a *stale* entry rather than any entry, e.g.:

  ```elixir
  stale? = Enum.any?(entries, &match?({:stale, _, _}, &1))
  if preload = stale? && opts[:preload], do: preload.()
  ```

## Risks / caveats

- I focused on paths reachable from the documented public API (`compile/6`, deprecated `compile/4`, `compile_entries/6`, `clean/1`, `outputs/1`, `diagnostics/1`, `to_erl_file/1`, `assert_valid_erlc_paths/1`). The deprecated `compile/4` and `ensure_application!/2` were reviewed but their legacy input shapes are contract-bound and not exercised further.
- I verified that `Path.absname/1` and `?/ in file` behave correctly on charlists (the Erlang-compiler diagnostic file format), so the `to_diagnostics/1` normalization is sound for its documented input.
- I did not attempt to trigger a raising compile callback under `Task.async_stream` to compare crash-propagation with the serial path; both simply propagate, matching expected "compile crashes on a raising callback" behavior, so I did not treat it as a distinct defect.

### Ruled out
- **`read_manifest/1` uses non-`:safe` `:erlang.binary_to_term`** — a crafted manifest could create atoms, but the manifest lives in the project's own `_build` directory (Mix-written, not peer/network input under this folder's threat model), and corruption is caught by the surrounding `rescue _ -> []`. Not attacker-reachable via the documented API.
- **`ensure_application!/2` `{:ok, _} = Application.ensure_all_started(app)` MatchError** — deprecated function; raising on a failed start is consistent with a bang-style contract, and the input is not untrusted.
- **`combine_results/2` reduce with `{:ok, [], [], []}` seed** — verified the accumulator and element are both 4-tuples in every branch of `do_compile/4`; ordering and concatenation are correct.
- **`lazy_modules_diff` `split_with` naming (`{changed, added}`)** — verified the predicate (`dest` present in `old_entries`) correctly maps `true → changed`, `false → added`; not a swap.

