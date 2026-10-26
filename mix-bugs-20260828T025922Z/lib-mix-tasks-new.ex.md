# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/new.ex
<!-- sha256: 9c4c02731256acb988f88ffc87d9fbf6307dd0704dba95186c13146fab88063f -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/new.ex
<!-- sha256: 9c4c02731256acb988f88ffc87d9fbf6307dd0704dba95186c13146fab88063f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

## Summary

`mix new` (`Mix.Tasks.New`) generates a new project skeleton from a PATH argument and a few options. On re-review I resolved the two hedged items from the first pass by executing them against the Elixir 1.21.0-dev build in this sandbox (Erlang/OTP 29): I drove the umbrella-detection path end-to-end inside a real `apps/` layout, and I ran `get_version/1`'s parse against `System.version()`. Both behave as the first pass assumed. No genuine behavioural bugs found.

## Bugfixes

No findings

## Risks / caveats

- `String.to_unsafe_atom/1` (line 212) is a real macro in this tree; it returns `:hello` and does not crash. `Module.concat(Elixir, "Foo.Bar")` (line 228) yields `Foo.Bar`, so dotted `--module` names resolve as intended (both re-confirmed against neighbour `Mix.Generator`, which uses `String.to_unsafe_atom` itself at generator.ex:220/249).
- `Mix.Generator.create_file/2` (generator.ex:36–53) calls `File.mkdir_p!(Path.dirname(path))`, so nested `lib/foo/bar.ex` and `test/foo/bar_test.exs` paths from `Macro.underscore/1` are created without error — read and confirmed in the neighbour module this pass, not assumed.

### Ruled out

- **`String.to_unsafe_atom(name)` atom creation (line 212)** — reached only after `invalid_app/1` passes (name matches `^[a-z][a-z0-9_]*$`), once per developer-invoked `mix new`, no loop over untrusted input. Not an exhaustion sink. Axes examined: reachability, loop/repetition, input trust source | Not examined: none.
- **`Module.concat(Elixir, name)` with dotted names (line 228)** — executed `mix new demo --module Foo.Bar` (exit 0) previously; the alias resolves to `Foo.Bar` and `check_mod_name_availability!/1` works for multi-segment modules. Axes examined: single-segment, dotted/multi-segment | Not examined: none.
- **`File.mkdir_p!(path)` when `path` names an existing regular file (line 81)** — raises `File.Error`; passing a file where a directory is expected is caller error outside the documented contract, and raising is the acceptable outcome. Axes examined: path-is-dir, path-is-file, path-absent | Not examined: none.
- **`check_directory_existence!/1` skipping the prompt for non-directory existing paths (lines 235–241)** — `File.dir?/1` is the correct guard for the "directory already exists" prompt; a same-named file is handled via downstream failure, consistent with the message shown. Axes examined: existing-dir (prompted), existing-file (not prompted, downstream failure), absent | Not examined: none.
- **`create_file`/`create_directory` parent-directory creation** — confirmed in `Mix.Generator` this pass: `create_file` does `File.mkdir_p!(Path.dirname(path))` (generator.ex:40). No missing-directory failure for nested module paths. Axes examined: flat path, nested path | Not examined: none.

### Resolved hedges (upgraded from hedged assumptions in first pass — now verified by execution)

- **Umbrella-detection path `in_umbrella?/0` (lines 253–267)** — first pass "did not exhaustively exercise" this inside a real umbrella. This pass I ran `mix new umbtest --umbrella`, then `mix new child_app` inside `umbtest/apps/`. The child's generated `mix.exs` contains `build_path: "../../_build"` (the `mix_exs_apps` umbrella-aware template), proving `in_umbrella?/0` returned `true` via the `Mix.Project.in_project(:umbrella_check, "../..", …)` branch. A standalone `mix new` outside an umbrella produces the plain `mix_exs` template, so the `false` branch is also exercised. The `catch _, _ -> false` is a deliberate best-effort probe defaulting to "not in umbrella"; not a defect. Confidence: N/A (ruled out — executed both branches).
- **`get_version/1` assumes `System.version()` always parses (lines 243–251)** — first pass assumed the `{:ok, version} = Version.parse(version)` match is infallible. This pass I ran it: `System.version()` → `"1.21.0-dev"`, `Version.parse/1` → `{:ok, %Version{major: 1, minor: 21, patch: 0, pre: ["dev"]}}`, and `get_version/1` → `"1.21-dev"`. The `[h | _]`/`[]` `pre` branch is exercised (pre-release build). `System.version()` returns the compiler's own version string, which is a valid SemVer by construction, so the match cannot fail on a real build. Confidence: N/A (ruled out — executed).

## Did I verify this?

Yes, by execution on the in-tree Elixir 1.21.0-dev / OTP 29 build:
- Umbrella detection: created a real umbrella and a child app inside `apps/`; verified the umbrella-aware template (`build_path`) was emitted, so `in_umbrella?/0` returned `true`; the standalone branch returns `false`.
- `get_version/1`: ran `Version.parse(System.version())` and the full `get_version` expression; confirmed the `{:ok, _}` match and the `pre` list branch.
- Neighbour `Mix.Generator` read directly (create_file/create_directory/overwrite?/embed_template) to confirm parent-dir creation and the shared `String.to_unsafe_atom` usage.

## Dropped in re-review

None. Every prior finding/ruled-out entry is preserved; the two prior hedged caveats (umbrella path, `get_version` parse) were moved to the "Resolved hedges" subsection with executed evidence rather than dropped.
