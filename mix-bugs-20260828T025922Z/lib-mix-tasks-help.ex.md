# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/help.ex
<!-- sha256: 1cc36e24094b14e1d47aa35b9a492b03da8d87577fb4c42bd46a391cc24005c9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:56:07Z_

## Summary

`mix help` is a display/introspection task; it is mostly straightforward, but the alias-rendering helper `format_alias_task/1` tests `Function.info/1`'s `:type` against the atom `:remote`, which that API never returns, producing wrong output for aliases defined as remote function captures.

## Bugfixes

#### `CSCAN-help-alias-fun-type-remote-typo`
- **Finding ID:** `CSCAN-help-alias-fun-type-remote-typo`
- **Severity:** Low
- **Confidence:** high
- **Location:** `elixir/lib/mix/lib/mix/tasks/help.ex:325`, `format_alias_task/1`
- **Class:** Missing clause / dead code (guard/`cond` clause that can never match)
- **Impact:** When a project defines an alias as a *remote* function capture (e.g. `aliases: [foo: &Enum.map/2]`), `mix help` and `mix help --aliases` render it as `Alias for &map/2` — the module qualifier is silently dropped. The intended output (via the `inspect(task)` branch) is `&Enum.map/2`. The first `cond` clause `info[:type] == :remote` is dead: `Function.info/1` returns `:external` (or `:local`), never `:remote`, so remote captures fall through to the generic `&#{name}/#{arity}` branch and lose their module. Cosmetic, but the help listing shows misleading/ambiguous alias documentation.
- **Evidence:** `verified`.
  - `Function.info(&Enum.map/2)[:type]` → `:external` (not `:remote`); `Function.info(fn x -> x end)[:type]` → `:local`.
    ```
    remote capture type: :external
    anon type: :local
    full remote info: [module: Enum, name: :map, arity: 2, env: [], type: :external]
    ```
  - Running the exact `cond` from lines 324–328 against `task = &Enum.map/2`:
    ```
    remote capture rendered as: "&map/2"
    ```
    while the intended `inspect(&Enum.map/2)` yields `&Enum.map/2`.
- **Suggested fix:** Change the first `cond` clause condition from `info[:type] == :remote` to `info[:type] == :external`.

## Risks / caveats
- I could not diff against the exact upstream Elixir source in-repo to confirm whether this typo is locally introduced or pre-existing upstream; either way the behaviour is a genuine defect (`:remote` is never a valid `Function.info` type value), verified by running against the `elixir` built from this tree.
- Other paths were reviewed and appear correct given Mix's default application env (`:mix, :colors` defaults to `[]`, so the `++ opts` in `ansi_opts/0` does not build an improper list in normal runs); a user who misconfigures `colors` to a non-list violates the documented `[colors: [...]]` contract and is out of scope.
- The `app:`/`t:`/`c:` introspection paths depend on `IEx.Introspection` and `Code.fetch_docs`; I validated their pattern shapes by reading, not by exercising every doc-chunk variant.

### Ruled out
- **`ansi_opts/0` `++ opts` improper-list crash** — Requires `:mix`/`:colors` env to be non-list; the folder's `mix.exs` sets `env: [colors: []]` (verified via grep), so the documented default is a list and no crash occurs for well-behaved callers.
- **`display_default_task_doc/0` with no project (`Mix.Project.get() == nil`)** — `function_exported?(nil, :cli, 0)` returns `false` rather than raising, so it correctly falls back to `"run"`.
- **`load_aliases/0` returning a map vs list into `build_alias_doc_list/1`** — both `--search` (list of tuples) and `--aliases`/no-arg (map) are iterated via `Enum.reduce`, which handles both; no shape mismatch.

