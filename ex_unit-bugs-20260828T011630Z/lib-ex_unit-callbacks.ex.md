# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/callbacks.ex
<!-- sha256: e4f6b18fb6bd181ea78207d95717578fe7b0b4ba65ad6fb29eb06f2166645091 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:22:50Z_

# Bug-Hunting Review — `elixir/lib/ex_unit/lib/ex_unit/callbacks.ex`

## Summary
A documented `setup`/`setup_all` invocation form — a list whose elements are all `{module, function}` tuples with a **plain-atom** module — is misclassified as a `do`-block by `Keyword.keyword?`, causing a confusing compile-time failure instead of registering the callbacks.

## Bugfixes

#### `CSCAN-callbacks-setup-atom-mod-tuple-list-as-block`
- **Finding ID:** `CSCAN-callbacks-setup-atom-mod-tuple-list-as-block`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `callbacks.ex:236-244` (`defmacro setup/1`) and `callbacks.ex:376-384` (`defmacro setup_all/1`), via the `Keyword.keyword?/1` dispatch.
- **Class:** Pattern match on the wrong shape (AST list of 2-tuples with atom key indistinguishable from a keyword/block list).
- **Impact:** A test author who writes the documented form "a list of atoms and `{module, function}` tuples" where every tuple's module is a plain atom (e.g. an Erlang module `:crypto`, or a module defined as `defmodule :my_mod`) gets a compile error `missing :do option in "defp"` pointing at the generated `__ex_unit_setup_N/1` function. The message names neither the real problem nor the offending `setup` line's intent, so the author cannot tell what to fix. The two common spellings that *do* work — a single `{Alias, :fun}` tuple, or a list containing at least one bare atom or an aliased module — mask how narrow the broken case is, making it look like a random ExUnit bug.
- **Evidence:** `verified`.
  - `setup/1` (line 237) routes to the block branch when `Keyword.keyword?(block_or_functions)` is true. At macro-expansion time an aliased module `MyMod` is an AST node `{:__aliases__, _, [...]}` (not an atom), so `[{MyMod, :f}]` is *not* a keyword list and works. But a plain-atom module stays an atom in the AST, so `[{:mod, :f}]` **is** a keyword list:
    ```
    $ elixir -e 'IO.inspect(Keyword.keyword?([{Mod1, :f1}, {Mod2, :f2}]))'   # alias form
    false
    ```
    (aliases → `false`, works), versus the atom-module list:
    ```
    setup [{:my_helper_mod, :step_a}, {:my_helper_mod, :step_b}]
    →  error: missing :do option in "defp"
       ** (CompileError) cannot compile module AtomModTest (errors have been logged)
    ```
    The identical failure reproduces for `setup_all [{:my_helper_mod4, :step_a}]`. Control cases confirm the boundary: `setup {:my_helper_mod3, :step_a}` (non-list tuple) → `1 passed`; `setup [{MyHelpers, :step_a}, {MyHelpers, :step_b}]` (aliased modules) → `1 passed`. The `setup` and `setup_all` macro bodies are byte-identical to the installed `/opt/elixir` build used to run these, so the reproduction reflects the reviewed source.
- **Suggested fix:** Distinguish "callback list" from "block keyword" more precisely than `Keyword.keyword?/1`. A `do`-block always arrives as `[do: ...]` (and optionally `:else`/`:catch`/`:after`), so gate the block branch on the presence of a `:do` key rather than mere keyword-ness — e.g. `if Keyword.keyword?(block_or_functions) and Keyword.has_key?(block_or_functions, :do)` in both `setup/1` and `setup_all/1` — routing everything else (including all-tuple lists) to `__setup__/__setup_all__`, whose `validate_callbacks!/1` already accepts `{module, function}` tuples.

## Risks / caveats
- Runtime lifecycle paths (`on_exit/2`, `start_supervised/2`, `stop_supervised/1`, monitor/link cleanup) were reviewed by reading only; they largely delegate to `ExUnit.OnExitHandler` and `Supervisor`, which live in other files and were out of scope. Cross-file invariants (e.g. `OnExitHandler.add` returning `:ok`/`:error`) were taken at their documented contract.
- The reproductions ran under a latin1-encoded VM (harness warning about native name encoding); this affects only console glyph rendering, not the compile-time macro dispatch under test.

### Ruled out
- **`__merge__` returning `{:ok, map}` / struct handling** — Verified: `{:ok, %{extra: 1}}` merges correctly (`1 passed`), and structs (`%_{}`) hit `raise_merge_failed!` as the docs promise ("Returning anything else … fail"). Behaves per contract.
- **`compile_setup/3` vs `compile_setup/2` arity confusion** — Read both call sites (line 191-192 pass `(kind, value, describes)`; line 835 passes `(callbacks, :setup)` to the 2-arity head; line 844 passes `(value, kind)`). Argument order is consistent with each head's parameter names; no transposition.
- **`stop_supervised/1` treating a missing child as success** — The `with :ok <- Supervisor.terminate_child(...)` short-circuits to `{:error, :not_found}` when the child is absent, and the `delete_child in [:ok, {:error, :not_found}]` guard is intentional (documented in the inline comment for temporary children). Matches the `@spec`.
- **Reserved-key merge only firing on keys present in both maps** (`context_merge`, line 756) — `Map.merge/3`'s function runs only for common keys, so a `setup_all` could set a reserved key absent from its context without raising. Left unreported: it matches long-standing upstream behavior and reserved keys are populated into the base context before setup runs; no divergence from the reviewed contract was demonstrated.

