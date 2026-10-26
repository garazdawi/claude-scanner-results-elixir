# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/dict.ex
<!-- sha256: 8ccdc914c2f91ab893e93dfd790087000d3135587cad4bbb00d82dd43bf73591 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:59:54Z_

## Summary

`Dict.empty/1` is dead-on-arrival for both documented dict types — it delegates to `Map.empty/1`/`Keyword.empty/1`, neither of which exists — so every call raises `UndefinedFunctionError`.

## Bugfixes

#### `CSCAN-dict-empty-undefined-delegate`
- **Finding ID** — `CSCAN-dict-empty-undefined-delegate`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `dict.ex:409` (`Dict.empty/1`, via `target(dict).empty(dict)`)
- **Issue** — `empty/1` resolves `target(dict)` to `Map` for a map and `Keyword` for a list, then calls `Map.empty/1` / `Keyword.empty/1`. Neither module exports an `empty/1` function (only `HashDict`, the legacy impl, ever defined one). The `@spec empty(t) :: t` with `@type t :: list | map` therefore names two input types for which the function can never succeed.
- **Class** — Missing clause / non-total function (delegation target does not exist).
- **Impact** — Any caller invoking `Dict.empty/1` on the only two types the module documents (a map or a keyword list) gets an `UndefinedFunctionError` rather than an empty dict. The function is unusable for its entire documented input domain, and the crash names `Map.empty`/`Keyword.empty` — functions the caller cannot define — giving no actionable path to fix.
- **Evidence** — `verified`.
  - `elixir -e 'IO.inspect(function_exported?(Map, :empty, 1)); IO.inspect(function_exported?(Keyword, :empty, 1))'` → `false` / `false`.
  - `Dict.empty(%{a: 1})` → `%UndefinedFunctionError{module: Map, function: :empty, arity: 1}`.
  - `Dict.empty([a: 1])` → `UndefinedFunctionError`.
- **Suggested fix** — This is deprecated code with no working target; the honest fix is to remove `empty/1` (and its `@spec`) from the public surface, or make it raise a clear `ArgumentError`/`RuntimeError` explaining that `empty` is unsupported and directing callers to construct an empty `Map`/`Keyword` directly, rather than bottoming out in an opaque `UndefinedFunctionError`.

## Risks / caveats

- The whole module is deprecated (`@moduledoc deprecated:`, per-function `@deprecated`), which lowers real-world exposure but does not change that the function is exported and documented.
- The `__using__`-generated functions (`get`, `merge`, `equal?`, `to_list`, etc.) were read but not reported: they are injected into a using module that must itself define `fetch/2`, `put/3`, `reduce/3`, `size/1`, `new/0`, and `delete/2`, and are correct given those (as in `HashDict`). Any defect there is reachable only through that internal contract, outside this review's public-API scope.

### Ruled out
- **`@spec fetch(t, key) :: value` mismatch** — `fetch` returns `{:ok, value} | :error`, not a bare value, but `@type value :: any`, so `{:ok, value}` and `:error` are both valid `any`; the spec is imprecise, not violated, and Dialyzer derives no error from it. Basis: `asserted` (spec logic) — return shape confirmed `verified` (`Dict.fetch(%{a:1}, :a)` → `{:ok, 1}`, `Dict.fetch(%{a:1}, :b)` → `:error`).
- **`do_merge` argument order to the merge fun** (`dict.ex:360`) — `fun.(k, other, v)` passes the dict1 value as `v1` and the dict2 value as `v2`, and the default `fn _k,_v1,v2 -> v2 end` makes the second dict win, matching `Map.merge/3`/`Keyword.merge/3` semantics. Basis: `asserted` (traced against the `merge/2,3` default fun and the delegated same-type path).
- **`Enumerable.reduce/3` result handling in `do_merge`/`equal?`** (`dict.ex:359,424`) — the raw `{:done|:halted, acc}` tuple is consumed with `|> elem(1)`, which correctly extracts the accumulator for both terminal shapes. Basis: `asserted`.
- **`merge`/`equal?` cross-type paths** — `target/1` classifies structs/maps/lists and routes unknown terms to `unsupported_dict/1` (a documented raise), so the reduce bodies only ever see `{k, v}` pairs from enumerable inputs. Basis: `asserted`.

