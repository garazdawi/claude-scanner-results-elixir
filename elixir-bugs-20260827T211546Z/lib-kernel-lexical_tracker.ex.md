# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/lexical_tracker.ex
<!-- sha256: 3dd001df35a57c7af964509ddd0aae13f31a3291d2ff21b595856f6eee7a9075 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:29:04Z_

## Summary

The file is a faithful gen_server implementation of the lexical tracker; the only defect found is a robustness asymmetry in the public `warn_import/2` path that crashes the whole tracker when its module was not previously registered via `add_import/5`.

## Bugfixes

#### `CSCAN-lexical_tracker-warn-import-nil-imports`
- **Finding ID** — `CSCAN-lexical_tracker-warn-import-nil-imports`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `lib/kernel/lexical_tracker.ex:277`, `handle_cast({:warn_import, module}, state)` (reached from public `warn_import/2`, line 46)
- **Issue** — The handler does `put_in(state.imports[module][@warn_key], true)`, a two-level nested update. If `state.imports[module]` has never been set (no prior `add_import`), the first level yields `nil` and the nested `put_in` raises `ArgumentError: could not put/update key 0 on a nil value`, terminating the gen_server. The sibling annotate functions are robust to this: `warn_require` uses a one-level `put_in(state.requires[module], …)` and `warn_alias` uses `Map.put`, both of which create the missing key rather than crash.
- **Class** — Missing clause / non-total function (nested `put_in` on `nil`)
- **Impact** — A caller invoking the documented `warn_import/2` for a module not first passed to `add_import/5` kills the tracker process; every subsequent `call` (`references`, `collect_unused_*`, `read_cache`, `stop`) from the compile job then fails with `:noproc`, aborting the compilation. In the actual compiler the crash is not reachable because `elixir_expand.erl:174-178` always emits `add_import` immediately before `warn_import` for the same module ref, so the invariant holds; the defect is the asymmetric fragility of a public function whose `@doc` never states the "must be imported first" precondition.
- **Evidence** — `verified`. Running `Kernel.LexicalTracker.warn_import(pid, NeverAdded)` on a fresh tracker:
  ```
  ** (ArgumentError) could not put/update key 0 on a nil value
      (elixir) lib/access.ex:438: Access.get_and_update/3
      ...
      lib/kernel/lexical_tracker.ex:277: Kernel.LexicalTracker.handle_cast/2
  alive after warn_import: false
  call failed: {:exit, {:noproc, {:gen_server, :call, [#PID, :unused_imports, :infinity]}}}
  ```
  Reachability of the safe path confirmed via grep: `elixir_expand.erl` pairs `add_import` (line 174/178) with `warn_import` (line 175) for the same `ERef`.
- **Suggested fix** — Guard against the missing outer map, mirroring the robustness of `warn_require`/`warn_alias`, e.g.:
  ```elixir
  def handle_cast({:warn_import, module}, %{imports: imports} = state) do
    case imports do
      %{^module => map} ->
        {:noreply, %{state | imports: Map.put(imports, module, Map.put(map, @warn_key, true))}}
      %{} ->
        {:noreply, state}
    end
  end
  ```

## Risks / caveats

- I reviewed the whole file clause-by-clause and drove every documented public entry point (`references/1`, `warn_require/4`, `warn_alias/4`, `warn_import/2`) plus the `@doc false` compiler-facing casts/calls through a live tracker (`/tmp/lt_test.exs`); outputs matched the intended semantics (partial-import bookkeeping, compile-vs-runtime reference partitioning, unused requires/aliases collection, cache round-trip, clean stop). No other crash, wrong-shape return, or accumulator-threading defect surfaced.
- The code matches current upstream Elixir semantics for `add_reference` (runtime never overwrites compile), `import_dispatch`/`import_quoted` (module sentinel + fa deletion), and `partition/3` (total over the only two values `add_reference` ever stores). The one finding above is also present upstream; it is reported here because it is a genuine crash on a public function, not because it diverges from upstream.
- The single finding is gated behind an invariant the real compiler always upholds, hence Low severity; it becomes reachable only if `warn_import/2` is called directly (e.g. from a test or an alternate front-end) without a preceding `add_import/5`.

