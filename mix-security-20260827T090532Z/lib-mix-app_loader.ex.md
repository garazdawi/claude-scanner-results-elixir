# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/app_loader.ex
<!-- sha256: 6769b1ee50ddfedab9bf9cff9086bcbff8545913b459f9b1f7c054f21f7cd747 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/app_loader.ex
<!-- sha256: 6769b1ee50ddfedab9bf9cff9086bcbff8545913b459f9b1f7c054f21f7cd747 -->

_Re-reviewed on 2026-08-27. All sink behaviour reproduced against the built-from-source OTP 29 / Elixir 1.21.0-dev toolchain; reachability edges verified by reading the four principal caller modules._

## Summary
`Mix.AppLoader` parses two kinds of untrusted, dependency-supplied artifacts — `.app` resource files (via `:erl_scan`/`:erl_parse`) and the compressed `compile.app_cache` manifest (via unsafe `:erlang.binary_to_term`) — and both parsers are unbounded, giving a dependency or a poisoned build cache a reliable denial-of-service against the build machine. Both findings retained from the first pass; the first-pass reachability hedge is now verified (see "Did I verify this").

## Security

#### `CSCAN-app_loader-app-file-atom-exhaustion`
- **Finding ID** — `CSCAN-app_loader-app-file-atom-exhaustion`
- **Severity** — Low
- **Confidence** — high
- **Location** — `app_loader.ex:68-69` (`read_app/2`, the `:erl_scan.string` / `:erl_parse.parse_term` calls); reached from `load_app/2` (line 84), `load_apps/5` → `traverse_apps` (line 132), and `Mix.Dep.Loader.app_status/3` (`dep/loader.ex:461`).
- **Class** — Atom creation (CWE-770) / Resource consumption.
- **Impact** — A malicious (including deep transitive) dependency ships a crafted `ebin/<app>.app` containing ~1M distinct atom literals (~20 MB). Whenever Mix loads the application tree or checks dependency status — `mix compile`, `mix run`, `mix deps`, `mix loadpaths`, `mix app.tree`, etc. — `read_app` runs `:erl_scan.string` over the whole file, creating one permanent (never-GC'd) atom per literal, exhausting the atom table and aborting the **entire BEAM** with `no more index entries in atom_tab`. This is a VM abort, not a catchable exception, so no `try/rescue` in Mix can recover it, and it fires even when the file is otherwise invalid — the atoms are created at scan time, *before* the `{:application, ^app, properties}` validation that would return `:invalid`. Net effect: a reliable, deniable, hard-to-diagnose DoS on developer/CI build machines from a data file. (Capped at Low because the dependency whose `.app` this is also executes code at compile time; the value here is that this crash is uncatchable, tree-wide, and triggers purely from parsing data.)
- **Evidence** — `verified`.
  - Sink creates one atom per identifier: scanning ~4 MB of `.app`-shaped text → `atoms created by scanning: 200001` (limit `1048576`).
  - End-to-end through the public function on a valid 6.5 MB `.app`: `Mix.AppLoader.read_app(:evil, "/tmp/evil.app")` → `read_app returned tag: :ok` / `atoms created by read_app: 300199`.
  - Actual whole-VM crash with the atom limit capped (`ELIXIR_ERL_OPTIONS="+fnu +t 100000"`): `read_app` → `no more index entries in atom_tab (max=100000)` + `Crash dump is being written`.
  - `:invalid` path is no protection: a malformed `.app` returned `:invalid` yet still produced `atoms created despite :invalid result: 100177`.
  - Reachability (verified this pass by reading callers): `compile.all.ex:43` calls `load_apps(apps, deps, config, …)` during every `mix compile`; `load_apps` → `traverse_apps` → `load_app` → `read_app` on each dep's `<app>.app` (`app_loader.ex:132`). `dep/loader.ex:461` calls `read_app` directly from `app_status/3` during `mix deps`. Both are on ordinary, non-privileged flows.
- **Suggested fix** — Bound the input before scanning: `stat` the file and refuse `.app` files above a small cap (legitimate `.app` files are a few KB) before calling `:erl_scan.string`. There is no drop-in "safe scan", so a byte-size ceiling on the `File.read` is the practical closure; alternatively tokenize with an explicit created-atom limit.

#### `CSCAN-app_loader-cache-unsafe-binary-to-term`
- **Finding ID** — `CSCAN-app_loader-cache-unsafe-binary-to-term`
- **Severity** — Low
- **Confidence** — high
- **Location** — `app_loader.ex:15` (`read_cache/1`, `:erlang.binary_to_term/1`); paired with `app_loader.ex:29` (`write_cache/2`, `term_to_binary(term, [:compressed])`). Called from `mix/compilers/elixir.ex:128`.
- **Class** — Deserialisation (CWE-502) + Resource consumption (decompression bomb).
- **Impact** — `read_cache` decodes `_build/.../compile.app_cache` with `:erlang.binary_to_term/1` **without `:safe`**, and the file is written `[:compressed]`. An attacker who can write that build artifact — a poisoned CI `_build` cache restored into a trusted job (a well-known GitHub-Actions cache-poisoning vector), a shared build server, or a committed `_build` — gets three DoS primitives at decode time, none requiring code execution: (a) atom-table exhaustion (unbounded new atoms), (b) a decompression bomb (`binary_to_term` transparently inflates the zlib payload with no size limit), and (c) an uncaught `TryClauseError` for any decoded term that isn't `{1, map}`, because the `rescue _ -> []` guards only the `try` body, not the `else` clause (`app_loader.ex:14-20`). No RCE results (the term is pattern-matched, never applied), so this is build-disruption/DoS.
- **Evidence** — `verified`.
  - Unsafe atom injection, fresh VM: a 261 KB compressed cache → `NEW atoms created by unsafe decode in fresh VM: 100030`; the same file with `[:safe]` raised `invalid or unsafe external representation` (so `:safe` blocks this vector).
  - Decompression bomb: `term_to_binary({1, %{a: <<0::300MB>>}}, [:compressed])` → `on-disk compressed cache size: 291621 bytes`, decode → `process memory grew by ~300 MB` (`amplification: ~1028x`); a few-MB manifest inflates to multiple GB → OOM.
  - Uncaught crash on non-matching term: a valid `term_to_binary(:garbage)` cache → replica of `read_cache` (the `try/rescue/else` at lines 14-20) returned `{:UNCAUGHT_RESCUE, TryClauseError}` (the `rescue` did not catch it, since the `else` clause runs outside the guarded body).
  - Reachability (verified this pass by reading the caller): `compilers/elixir.ex:128` calls `Mix.AppLoader.read_cache()` during the Elixir compiler pass of `mix compile`.
- **Suggested fix** — Treat the manifest as untrusted: decode with `[:safe]` and move the `{@manifest_vsn, _}` shape match *inside* the guarded region (or a `case`) so a hostile/malformed term falls through to `[]` instead of raising. Cap the on-disk manifest size before reading to blunt the decompression bomb (`binary_to_term` offers no decompressed-size limit). Note the trade-off: `[:safe]` raises on atoms not yet in the table, so if the cached module atoms may be absent at read time it will fall back to `[]` (a full recompile, not a crash) — acceptable, but the size cap + guarded shape-check is the lower-regression core of the fix.

### Ruled out

- **Path traversal via a hostile `.app` `:applications` atom** (e.g. `{applications, ['../../evil']}`).
  - Reasoning: a hostile atom loaded via `:application.load` and surfaced by `app_children` (line 156) enters the `traverse_apps` worklist, but it is not a key in `deps_children` (keys are real `dep.app` atoms) so it takes the `_ -> {builtin_paths[app], []}` branch (line 124); `builtin_paths[hostile]` is `nil`, so `with true <- ebin_path != nil` (line 131) fails and the app is dropped with `[]` before any file operation. `app_join` (line 159) is only reached with `app` that is a `deps_children` key (line 123) or a real builtin name — never with the injected atom. The atom is still *created*, which is covered by the atom-exhaustion finding.
  - **Axes examined:** control-flow dispatch of a non-dep, non-builtin atom through `traverse_apps`/`app_join` (by reading lines 114-159). **Not examined:** executing a crafted `../`-bearing atom end-to-end through `traverse_apps` (the function is private and the worklist is fed only from `Application.spec`, so I did not drive a hostile atom through it in a live VM). Entry remains `asserted`.

- **`stale_cache/1` path interpolation** `Path.join([opts[:build], "ebin", "#{app}.app"])` (line 50).
  - Reasoning: the interpolated path feeds only `Mix.Utils.last_modified` (a `stat`); `app`/`opts[:build]` come from resolved dep specs (`Mix.Dep.cached()`, line 48). No write/read/exec sink, negligible info exposure.
  - **Axes examined:** sink kind (stat-only), provenance of `app`/`opts[:build]` (resolved dep structs, by reading lines 47-54). **Not examined:** whether a dep spec can carry a `../`-bearing `opts[:build]` that a `stat` would traverse (I did not construct a hostile resolved-dep struct and run `stale_cache` against it). A `stat` has no write/exec effect regardless, so even an untried traversal here is information-grade only. Entry remains `asserted`.

- **`load_app/2` → `:application.load` of attacker-controlled `properties`** (line 85).
  - Reasoning: `:application.load` registers config but never invokes the `:mod` callback; app identity is validated by the `^app` match in `read_app` (line 69), and `:env` applies only to the dependency's own app — no cross-app poisoning, no execution.
  - **Axes examined:** `:application.load` semantics (config registration vs. `:mod` start), the `^app` identity pin. **Not examined:** whether a crafted `:env`/`:included_applications` property can influence a *different* app's environment through a live `:application.load` call (I reasoned from documented `load` semantics but did not run a cross-app `:env` poisoning attempt). Entry remains `asserted`.

- **`opts_app` as an arbitrary `.app` path** (`dep/loader.ex:407`): the binary path comes from the project's own `mix.exs` `:app` option (operator-controlled), not from a dependency/network/archive — out of the attacker model.
  - **Axes examined:** provenance (operator `mix.exs`). **Not examined:** none required — the source is operator-controlled config, which the folder threat model places outside the boundary by design. Entry is a scoping exclusion, not a suppressed sink.

## Risks / caveats
- **Trust-boundary honesty**: both findings are gated behind an actor who already has significant reach — the `.app`-exhaustion actor is a dependency that also runs code at compile time, and the cache-poisoning actor needs write access to `_build` (where they could also drop a `.beam`). That is why both are rated Low despite being clean, verified, whole-VM DoS conditions. Their residual value is the CI-cache-restore and prebuilt/transitive-dep scenarios where the DoS lands *before or without* the actor's code executing, and the fact that the atom-table crash is uncatchable.
- **Minor, not separately reported**: `read_app` calls `String.to_charlist(bin)` (line 68) with no rescue, so an `.app` file containing invalid UTF-8 raises `UnicodeConversionError` up through `load_app`/`traverse_apps`/`load_apps`. This is a strictly weaker crash than the atom-exhaustion finding (a single catchable-at-top-level failure vs. a VM abort) from the same input source, so it is folded into that finding rather than listed on its own.

## Did I verify this
- **Both findings' sink behaviour** — `verified` by execution (atom counts, VM abort with capped atom table, `[:safe]` rejection, 1028× decompression amplification, `TryClauseError` escaping `rescue`), as in the first pass; re-confirmed against the unchanged file (sha256 matches).
- **Reachability of `read_app`/`load_apps`** — **verified this pass** by reading the callers (previously hedged as "did not exhaustively enumerate every Mix task"). `tasks/compile.all.ex:43` invokes `load_apps` on every `mix compile` (traced through `do_run`, lines 27-47), which reaches `read_app` per dep `.app` at `app_loader.ex:132`; `dep/loader.ex:461` invokes `read_app` directly from `app_status/3`; `tasks/compile.ex:161` is a second `load_apps` entry point. This is a call-edge claim, not a guard-disagreement axis, so reading the caller source is sufficient verification — and it upgrades the first-pass hedge to a confirmed edge. I did not enumerate *every* transitive task, but the confirmed edges establish that both sinks fire during ordinary `mix compile` and `mix deps`, which is all the findings claim.
- **Reachability of `read_cache`** — **verified this pass**: `compilers/elixir.ex:128` calls it during the Elixir compiler pass; `stale_cache`/`write_cache` are the companion producers in `compile.all.ex:32,83`.
- **Ruled-out entries** — each carries an `Axes examined | Not examined` field above. None were promoted to `verified`: for each, at least one axis (driving a hostile value end-to-end through the private `traverse_apps`/`stat`/`:application.load` path in a live VM) was not executed, so per the downgrade-preferring rule they remain `asserted` dismissals rather than verified ones. Reading the neighbour modules (`dep/loader.ex`, `tasks/compile.all.ex`) informed the control-flow reasoning but did not, by itself, clear any untried axis.

### Dropped in re-review
None. Both findings are retained with their original executed evidence; no dismissal was upgraded to `verified` and no finding was dropped.
