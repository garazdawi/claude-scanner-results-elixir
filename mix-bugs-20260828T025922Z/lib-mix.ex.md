# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix.ex
<!-- sha256: 46953183c70cecd77c32d9ff427de86bdf2f571e031506879eacd2a8c937f1a0 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:05:23Z_

## Summary

`mix.ex` is the `Mix` core module (env/target/shell/debug state accessors, `raise`, `install/2`, `ensure_application!`). The runtime logic is sound; the defects found are contract/type mismatches around `Mix.install/2`'s option surface rather than reachable crashes.

## Bugfixes

#### `CSCAN-mix-install-opts-missing-stop-unused`
- **Finding ID:** `CSCAN-mix-install-opts-missing-stop-unused`
- **Severity:** Low
- **Confidence:** high
- **Location:** `mix.ex:432-444` (`@type install_opts`), used by `@spec install/2` at `mix.ex:874-877`
- **Class:** Contract violation (`@spec` vs implementation)
- **Impact:** `:stop_unused_applications` is a documented option (`mix.ex:761-763`) accepted by `Keyword.validate!` (`mix.ex:923`), but it is absent from the `install_opts` type that `install/2`'s `@spec` references. A caller who follows the docs and writes `Mix.install(deps, stop_unused_applications: false)` runs correctly at runtime but is flagged by Dialyzer as passing an option outside the declared type. The type error then propagates into every caller's static analysis.
- **Evidence:** asserted — read `mix.ex:432-444` (the type lists `force, verbose, consolidate_protocols, elixir, system_env, config, config_path, lockfile, start_applications, compilers, runtime_config`; no `stop_unused_applications`), against the documented option at `mix.ex:761` and the `Keyword.validate!` allow-list at `mix.ex:923`.
- **Suggested fix:** Add `stop_unused_applications: boolean()` to the `install_opts` type.

#### `CSCAN-mix-install-runtime-config-ignored`
- **Finding ID:** `CSCAN-mix-install-runtime-config-ignored`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `mix.ex:443` (type) and `mix.ex:921` (`Keyword.validate!` default), in `install/2`
- **Class:** Documentation mismatch (typed option that has no effect)
- **Impact:** `:runtime_config` appears in the `install_opts` type and is accepted/defaulted by `Keyword.validate!`, but the value is never read anywhere in `install/2` (or elsewhere in Mix). A caller passing `runtime_config: [...]` — a key the published type invites — gets no effect and no error, silently losing whatever configuration they intended.
- **Evidence:** verified (grep) — `runtime_config` in `elixir/lib/mix/lib/mix.ex` occurs only at line 443 (type) and line 921 (validate default); it is never fetched from `opts` (the post-validate code fetches `config, system_env, consolidate_protocols, start_applications, compilers, verbose, stop_unused_applications, force, config_path, lockfile` only). No other file in `lib/mix` reads an install-level `runtime_config`.
- **Suggested fix:** Either consume `:runtime_config` where intended, or remove it from both the type (`mix.ex:443`) and the `Keyword.validate!` defaults (`mix.ex:921`) so the option surface reflects reality.

## Risks / caveats

- I could not run Dialyzer to observe the type error directly; the `stop_unused_applications` finding is a static reading of the type against the documented/validated option set, so its evidence is `asserted` (though the discrepancy is plain in the source).
- `Mix.install/2`'s deeper behaviour (deps.get/compile/app.config orchestration, `stop_apps`, restore-dir handling) depends on many other modules (`Mix.ProjectStack`, `Mix.Task`, `Mix.Dep.Lock`) and process-global state; I reviewed it for local logic errors but did not execute a full install, which requires network/dep fetching outside this sandbox.
- The `silence_app_exit/2` logger-filter pattern (`report: [application: _, exited: :stopped] ++ _`) relies on OTP's fixed application-exit report key order; it is a private function reachable only through `install/2` and matched the current OTP report shape on review, so it is not reported.

### Ruled out
- **`raise/2` with `exit_status: 0`** — `opts[:exit_status] || ...` looks like it would discard a `0`, but `0` is truthy in Elixir (only `nil`/`false` are falsy), so `exit_status: 0` is honored. No bug.
- **`Mix.install(deps, force: true)` called a second time in the same VM raises "same dependencies" error** — the `{^id, _} when not force?` guard falls through to the generic raise when `force?` is true. This is the documented/known limitation that a VM cannot re-install once deps are code-loaded (the `force`/`rm_rf` path is designed for a fresh VM whose on-disk cache exists but whose `:installed` state is `nil`); not a defect in this file's contract.
- **`shell/1` doctest (`iex> Mix.shell(Mix.Shell.IO)` → `:ok`)** — `Mix.State.put/2` returns `:ok` (`state.ex:36-39`), so the doctest is accurate; and no test module runs `doctest Mix` (only `Mix.Utils`/`Mix.Release`), so the un-runnable `iex> Mix.install([])` example inside `install/2`'s `@doc` is never executed as a test.
- **`ensure_application!/1` threading of the `optional` accumulator into recursion** — traced required/optional propagation across nested builtin apps; missing *required* deps still raise and missing *optional* deps are correctly skipped at each level. Behaves as intended.
- **`expand_path/4` atom clause ordering** — `nil` is handled by the first clause before the `is_atom/1` clause, so an atom `config_path`/`lockfile` naming a path dep resolves correctly; non-path or unknown atoms raise per contract.

