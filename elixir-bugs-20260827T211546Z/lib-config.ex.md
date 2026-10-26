# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/config.ex
<!-- sha256: aa996f3c0d5de7977483af51cd5a1e6069ff3fe1e004f82f067c03826b00d58c -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:58:03Z_

## Summary

`Config` is a small, well-behaved keyword-configuration module; the only defect found is a documentation-vs-code mismatch in the `config/3` `@doc` example (wrong result ordering), with no reachable runtime bug in the public API.

## Bugfixes

#### `CSCAN-config-config3-doc-order-mismatch`
- **Finding ID** — `CSCAN-config-config3-doc-order-mismatch`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `lib/config.ex:192-198`, `@doc` example for `config/3`
- **Issue** — The documented `#=>` result for the merged `:ecto`/`Repo` configuration is `[log_level: :info, pool_size: 10, adapter: Ecto.Adapters.Postgres, metadata: [read_only: true, replica: true]]`, but the actual deep-merge (`Keyword.merge` preserving first-list key positions and appending new keys) produces `[adapter: Ecto.Adapters.Postgres, log_level: :info, pool_size: 10, metadata: [read_only: true, replica: true]]`. The set of keys, which value wins on conflict, and the nested `metadata` deep-merge are all correct — only the key *order* shown is impossible to produce.
- **Class** — Documentation mismatch
- **Impact** — A reader who takes the documented order literally (e.g. code that pattern-matches or `Enum.at/2`-indexes the resulting keyword list, or that visually diffs config) is misled. No crash; keyword order is generally not semantically meaningful for application env, so cost is limited to confusion. This is a descriptive `#=>` example, not an executable `iex>` doctest, so it does not fail the doctest suite.
- **Evidence** — `verified`. Driving the real public path:
  ```
  Config.config(:ecto, Repo, log_level: :warn, adapter: Ecto.Adapters.Postgres, metadata: [read_only: true])
  Config.config(:ecto, Repo, log_level: :info, pool_size: 10, metadata: [replica: true])
  # => [adapter: Ecto.Adapters.Postgres, log_level: :info, pool_size: 10,
  #     metadata: [read_only: true, replica: true]]
  ```
  which differs from the documented order (`log_level, pool_size, adapter, metadata`).
- **Suggested fix** — Update the doc block to the order the code produces (`adapter:` first, then `log_level:`, `pool_size:`, `metadata:`), or reword the example to state that key order is unspecified.

## Risks / caveats

- The two macros (`config_env/0`, `config_target/0`) and `import_config/1` expand into `Config.__*__!/1` helpers and rely on `__DIR__`/`Code.eval_string` behaviour that is exercised only when a real config file is evaluated through `Config.Reader`; I reasoned about that flow but did not stand up a full `Config.Reader` evaluation, so the macro-expansion paths are `asserted`, not end-to-end `verified`.
- The `unicode`/tokenizer standards checklist supplied in the standing context does not apply to this file — `Config` performs no Unicode identifier, script, or normalization work — so none of those clauses were adjudicated here.

### Ruled out
- **`config/2` and `config/3` return the *previous* config, not the new merged config** (`lib/config.ex:163,206`): `put_config/1` is `Process.put/2`, which returns the prior value. Verified: `Config.config(:logger, level: :info)` returned `[]` while the stored config became `[logger: [level: :info]]`. Not a bug in scope — the `@spec ... :: keyword()` is still satisfied (the pdict is initialized to `[]` in `__eval__!`, so the return is always a keyword list, never `nil`), the return value is undocumented, and config files discard it. Basis: `verified`.
- **`env: false` / `target: false` makes `config_env()`/`config_target()` raise "no :env key was given"** (`lib/config.ex:259,282`): `false` satisfies `is_atom/1`, and `elem(opts, 0) || raise` treats it as absent. Not reachable through a documented public path — `env`/`target` are supplied internally by `Config.Reader` from `MIX_ENV`/`MIX_TARGET` and are never the literal atom `false`; it also requires input violating the ordinary `env: atom()` intent. Basis: `asserted` (behaviour of `||` on `false` is language-level certain; unreachability argued from the internal-only `__eval__!` entry point).
- **`validate!/2` second (non-list) clause is dead** (`lib/config.ex:401-405`): its only caller (`lib/config.ex:333`) is guarded by `is_list(eval_config)`, so the non-list clause never runs. It is a private, defensive fallback, not a defect, and is out of scope (private function). Basis: `verified` (guard at line 332 gates the sole call site).
- **`after`-block restore uses truthiness (`if previous_x, do: put, else: delete`)** (`lib/config.ex:339-341`): safe here because none of the saved values (`{env,target}` tuple, config keyword list including `[]`, imports list/`:disabled`) can be the falsy atom `false`, and `[]` is truthy in Elixir; `nil` correctly means "no prior key → delete". Basis: `verified` (`[]` truthiness and `Process.put` returning `nil` when absent both confirmed by running).
- **`import_config!` recursion guard / `:disabled` handling** (`lib/config.ex:345-366`): the `cond` covers `:disabled` (raise or silently skip based on `raise_when_disabled?`), already-imported (raise), and new (record + eval); expanded paths are compared consistently. No missing shape found across the three branches. Basis: `asserted` (path analysis of all three `cond` arms).

