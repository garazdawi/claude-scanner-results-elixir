# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/parallel_checker.ex
<!-- sha256: bbb5ae467747a4f014f29d7e546d02323ccd8d5d982bf2d8915618ee2586f713 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:44:11Z_

# Review: `Module.ParallelChecker`

## Summary
The file is a tightly-integrated compiler-internal gen_server whose concurrency/message-flow logic is sound; the one defensible finding is a `@spec` on the documented `fetch_export/5` that omits a signature shape the code can actually return.

## Bugfixes

#### `CSCAN-parallel_checker-fetch-export-spec-strong-sig`
- **Finding ID:** `CSCAN-parallel_checker-fetch-export-spec-strong-sig`
- **Severity:** Low
- **Confidence:** high
- **Location:** `parallel_checker.ex:235-236` (`@spec fetch_export/5`), reached from `parallel_checker.ex:258` and populated at `parallel_checker.ex:504`.
- **Issue:** The `@spec` declares the returned signature element as `{:infer, [term()] | nil, [term()]} | :none`. But `cache_chunk/3` stores any sig whose tag is in `[:infer, :strong]` (`%{sig: {key, _, _} = sig} when key in [:infer, :strong] -> sig`, line 504), and `fetch_export/5` hands that stored tuple straight back as the 4th element (`[{_key, reason, signature}] -> {:ok, mode, reason, signature}`, line 258). A `{:strong, _, _}` tuple is therefore a legitimate return value that the `@spec` does not admit.
- **Class:** Contract violation (`@spec` disagreeing with implementation).
- **Impact:** Dialyzer builds a return type for `fetch_export/5` narrower than reality; every caller that pattern-matches or guards on a `{:strong, …}` signature will be flagged as an impossible/failing match, and the noise propagates into all downstream analyses even though the runtime behaviour is correct. No runtime failure.
- **Evidence:** asserted — read from source: the `:strong` tag is inserted at line 504 (confirmed by grep) and returned verbatim at line 258, while the spec at lines 235-236 lists only `:infer`. A full confirmation would require a Dialyzer run over a module whose ExCk chunk carries a `:strong` sig; the code paths themselves are unambiguous.
- **Suggested fix:** Widen the spec's signature element to cover the strong tag, e.g. `{:infer | :strong, [term()] | nil, [term()]} | :none` (or introduce a shared `@type signature ::` used by both `cache_chunk` producers and this spec).

## Risks / caveats
- The module is `@moduledoc false` and every function is compiler-internal; the entry points (`start_link/1`, `verify/1,2`, `spawn/6`, `fetch_export/5`, `get/0`, `put/2`, `stop/1`) carry preconditions established by their compiler callers (a set `:elixir_checker_info`/`:elixir_compiler_info` pdict, well-formed module maps, arity-1/2 callbacks). Several functions will raise `CaseClauseError`/`FunctionClauseError` when those preconditions are violated, but those are contract requirements, not reachable public-API bugs — see Ruled out.
- I did not attempt to drive the gen_server end-to-end (it requires a live compiler session with real ExCk chunks); the concurrency reasoning below is static.

### Ruled out
- **`get/0` `CaseClauseError` on `:undefined` pdict (line 47):** the two clauses handle `{parent, nil}` and `{parent, cache}` but not `:undefined`. Reachable only if a caller invokes `get/0` before `verify/1` seeds `:elixir_checker_info`; that ordering is a documented internal precondition (the pdict is always set by the surrounding compile driver), so this is contract, not a public-path defect. Basis: asserted (read of the only two writers of the key, `verify/1:154` and `put/2:62`).
- **`init/1` `each_long_verification` `CaseClauseError` (lines 576-580):** falls through if the option is not an arity-1 or arity-2 function, but the `start_link_opts` type documents exactly those two arities; passing anything else violates the declared contract. Basis: asserted (matched against the `@type` at lines 19-25).
- **`merge_compiler_no_warn_undefined/1` `CaseClauseError` (lines 338-343):** crashes if `Code.get_compiler_option(:no_warn_undefined)` is neither `:all` nor a list; that option is compiler-owned and only ever set to those shapes, so it is not reachable through this file's API. Basis: asserted.
- **`verify/2` result count / mailbox hang (lines 198-228):** I traced that every registered cache — whether the runtime-file path (`{module, pid, ref}` → one `{__MODULE__, module, …}` reply from `spawn_checker`/the else-branch at line 133) or the compile path (`{module, mode}` with its pre-spawned checker) — produces exactly one decrementing message, so `count` from `:start` matches the number of `collect_results` decrements. No off-by-one or lost-reply found. Basis: asserted (static message-flow trace across `spawn_and_register_cache`, `run_caches`, `spawn_checker`, `handle_call(:start)`).
- **`lock`/`unlock` `Map.fetch!` on missing module (line 617):** `unlock` runs only inside `if lock(...) do … end` (lines 427-460), and a `true` lock reply always inserts `waiting[module] = []` first (line 610-611), so the key is present; waiters that receive `false` never call `unlock`. No double-unlock. Basis: asserted (serialized through the single gen_server; both call sites read).
- **`:checked` timer cancel with `nil` (line 645):** `timer && Process.cancel_timer(timer)` correctly no-ops for cache entries stored as `ref => nil`. Basis: asserted.

