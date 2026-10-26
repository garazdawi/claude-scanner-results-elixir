# Cluster cross-file findings

#### `CSCAN-cluster-reserved-context-keys-diverge`

- **Finding ID** — `CSCAN-cluster-reserved-context-keys-diverge`
- **Severity** — Low
- **Confidence** — high
- **Files involved** —
  - `elixir/lib/ex_unit/lib/ex_unit/callbacks.ex:711` (`@reserved [:case, :file, :line, :test, :async, :registered, :describe]`) and its use in `context_merge/4` at `callbacks.ex:756-764`
  - `elixir/lib/ex_unit/lib/ex_unit/case.ex:351` (`@reserved [:module, :file, :line, :test, :async, :registered, :describe]`), its use in `validate_tags/1` at `case.ex:910-920`, and the documented "known/reserved" key list at `case.ex:161`
  - Corroborating (out-of-cluster, folder): `elixir/lib/ex_unit/lib/ex_unit/runner.ex:293,322` where the actual test context is built.
- **Finding shape** — State-machine / inherited-assumption inconsistency (an invariant — "framework-owned context keys are protected from user overwrite" — set in one file and violated by the divergent list in another).
- **Evidence** —
  - The runner builds the per-test context with `:module` (not `:case`): `runner.ex:293` `module: test.module` and `runner.ex:322` `context = tags |> Map.merge(params) |> Map.merge(%{module: module, async: async?})`. There is no `:case` key placed into the context anywhere (`:case` survives only as a deprecated `%ExUnit.Test{}` struct field — `case.ex:727`, and `ex_unit.ex:170` `# TODO: Remove the :case field on v2.0`).
  - `case.ex` correctly treats `:module` as reserved: `@reserved [:module, ...]` (`case.ex:351`), enforced in `validate_tags/1` (`case.ex:911`), and documents `:module` as an ExUnit-reserved key (`case.ex:161`). It does **not** list `:case`.
  - `callbacks.ex` uses the opposite list: `@reserved [:case, :file, :line, :test, :async, :registered, :describe]` (`callbacks.ex:711`). In `context_merge/4` (`callbacks.ex:756-764`), only keys in that list are guarded: `k, v1, v2 when k in @reserved -> ... raise_merge_reserved!`; everything else falls through to `_, _, v -> v` (silent overwrite).
  - Net effect: a `setup`/`setup_all` callback that returns `%{module: SomethingElse}` (or `[module: ...]`) is silently merged over the framework-set `:module` context entry — the exact clobbering `context_merge`'s reserved check exists to prevent — while the check instead defends `:case`, a key that is never present in the context, so that arm is dead code.
- **What the per-file pass missed** — A single-file reviewer of `callbacks.ex` sees a self-consistent `@reserved` list and a working guard; only by comparing it against `case.ex`'s reserved list *and* the context actually produced in `runner.ex` does it become clear the list guards a phantom key (`:case`) and omits a live one (`:module`).
- **Suggested fix** — Align `callbacks.ex:711` with the context keys / `case.ex`: replace `:case` with `:module`, i.e. `@reserved [:module, :file, :line, :test, :async, :registered, :describe]` (or add `:module` if `:case` must be kept for legacy tolerance until the v2.0 field removal).
