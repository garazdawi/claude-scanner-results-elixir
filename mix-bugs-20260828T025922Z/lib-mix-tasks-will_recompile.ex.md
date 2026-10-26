# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/will_recompile.ex
<!-- sha256: 1b265ba90baa9e74b03fdd3a41184966f0e292a26c9b36b0387324b31a5d2bf1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T05:08:12Z_

## Summary

`will_recompile.ex` is a tiny internal Mix task (`@moduledoc false`) whose `run/1` touches a `.mix/<manifest>` marker file under the build path(s) to signal that the project should recompile; the logic is straightforward and I found no genuine defects.

## Bugfixes

No findings

## Risks / caveats

- The file's only public surface is `run/1` (the `Mix.Task` behaviour); the module itself is `@moduledoc false`. I reviewed the whole file (41 lines) and traced its collaborators: `Mix.ProjectStack.reset_config_mtime/0` (returns the `@manifest` binary — a valid filename, confirmed at `project_stack.ex:126-134`), and `Mix.Project.{umbrella?,build_path,apps_paths,app_path}`. In the umbrella branch `apps_paths/1` is guaranteed non-nil because `umbrella?/1` already returned true, so the `Enum.map` over its map is safe.
- I did not execute the task end-to-end (it mutates the build directory of a real project and depends on a populated `ProjectStack`); the assessment rests on static reading plus confirmation of the return shape/`@spec` of `reset_config_mtime/0`. The map-enumeration, path-join, and `File.mkdir_p!/touch!` calls have no error-path divergence from their documented contracts.

### Ruled out

- **Umbrella root vs. child path split** (`build` vs `build/lib/<app>`) — deliberate, matching the in-code comment about needing manifests for both root and children; each `path` is independently `mkdir_p!`'d before `touch!`, so no missing-directory crash.
- **`apps_paths/1` returning `nil`** — only inside the `umbrella?/1`-true branch, where `apps_paths/1` returns a map by contract; `Enum.map` never receives `nil`.
- **`reset_config_mtime/0` return value** — its `@spec` is `binary` and it returns `@manifest` (a static filename string), so `Path.join(path, filename)` always gets a valid binary; no shape mismatch.

