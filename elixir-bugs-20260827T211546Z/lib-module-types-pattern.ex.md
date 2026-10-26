# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/pattern.ex
<!-- sha256: a3db1a42293c4699098e4880c7e961a427975dcfcfbbf141c64db52bcb2bd512 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:27:24Z_

## Summary

After reading the module in full and driving its public entry points (`of_head`, `of_match`, `of_generator`, `of_guard`, `of_match_var`, `format_diagnostic`) through the compiler's type checker with a broad range of patterns and guards, I found no genuine, reachable bug.

## Bugfixes

No findings.

## Ruled out

- **`of_pattern_tree({:intersection, entries})` (line 562) reducing an empty list via `Enum.reduce/2`** — `Enum.reduce/2` raises on `[]`, but the only producer is the `:=` clause (lines 680–698), where the `dynamic` accumulator is seeded with `[{:var, version}]` and never emptied, so `entries` is always non-empty. Basis: `verified` — nested matches `x = y = z`, `x = 1 = y`, `x = %{a: 1} = %{b: 2}` all compile and type-check without error.
- **`args_to_previous/1` (lines 32–33) having no `[]` clause → `FunctionClauseError`** — every call site guards against `[]` first (`concat_previous([], …)` line 26; `previous_subtype?([], _)` line 23; `previous_subtype?(_, {[], _})` line 22). Basis: `verified` — single-arg and multi-arg redundant-clause detection (which flows args_types through `concat_previous`/`previous_subtype?`) emits the redundancy warning cleanly instead of crashing.
- **`of_head/8` returning inconsistently-shaped tuples** — the initial destructure takes the 5-tuple from `of_precise_head` (`{trees, precise?, check_previous?, args_types, context}`), while every `of_head` return is the distinct 5-tuple `{trees, precise?, args_types, previous_or_concat, context}`; all branches (lines 195, 200, 203, 210, 213, 216) agree with each other. Basis: `asserted` (structural read); consistent across all return paths.
- **`of_subpattern/4` integer key collision (line 991, `key = map_size(subpatterns)`)** — `subpatterns` also holds non-integer keys (`{:list, v}`, `{fun, arg_key}`), but integer keys are assigned as the map size at insertion time and the map only grows, so each new integer key strictly exceeds all prior ones — no collision. Basis: `verified` — list head/tail subpatterns, `hd`/`tl` guards, and domain-key maps combined in one module compile without error.
- **`format_diagnostic({:redundant, …})` arity/shape mismatch between the two clauses (lines 1531, 1555)** — the 3-tuple `def` tag routes to the first clause and the 2-tuple tags (`{info, args}`) to the second; the inner `with {_op, meta, expr, type} <- info` correctly falls through to the generic message for atom-headed tags like `:for_reduce`. Basis: `verified` — redundant `def`, `case`, and `x || …` diagnostics all render the expected messages.
- **General crash-on-input across the pattern/guard surface** — exercised atoms/ints/floats/binaries, `%mod{}`, `%_{}`, sized bitstrings with dependent sizes, `[h|t]`/`[a,b|c]`/`[…]++rest`, empty tuple, chained `=`, domain-key maps, `x.field` guards, `not is_struct/2`, `x == y` var comparisons, `x in […]`, `hd/tl` guards, comma/`when`-`when` guards, and `or`/`and` chains. Basis: `verified` — none produced a compiler crash; warnings matched the code paths.

## Risks / caveats

- The module is `@moduledoc false` with no `@spec`s or doctests, so there is no documentation/spec contract to contradict; the reachable surface is "compile Elixir source → type checker runs."
- My behavioral coverage is broad but not exhaustive of every internal branch (e.g. `of_changed`/`of_changed_deps` dependency-propagation fixpoint, and the `orelse … :fail` / `reconstruct_lists_member` rewriting were only lightly driven). I found no evidence of a defect in those paths on reading, but I did not construct an input that provably forces every intermediate state, so a subtle wrong-type or wrong-message bug deep in the dependency-refinement loop cannot be fully excluded from what I ran.

