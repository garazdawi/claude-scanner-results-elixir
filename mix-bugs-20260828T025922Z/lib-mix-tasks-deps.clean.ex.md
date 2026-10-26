# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.clean.ex
<!-- sha256: 11d176c0580f1bf3b22e4a2da8b8fb0183f85eff6d200dc082250b05097af30e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:47:35Z_

The file is byte-identical to the reference Mix 1.21.0-dev source. Combined with my behavioral checks, I have enough to report.

## Summary

`Mix.Tasks.Deps.Clean` implements the `mix deps.clean` task (public entry `run/1`); after enumerating and testing the plausible defect sites (comprehension filtering, arg-forwarding to `deps.unlock`, nested lock re-entrancy, string/atom type consistency, and `File.rm_rf` error handling), I found no genuine reachable bug — the file is byte-identical to the canonical Elixir source.

## Bugfixes

No findings

## Risks / caveats

- I could not run `mix deps.clean` end-to-end against *this* tree: the checkout only contains `lib/mix`, and the on-PATH toolchain is a separate `/opt/elixir` 1.21.0-dev install. However, `deps.clean.ex` in this tree is **byte-identical** to `/opt/elixir/.../deps.clean.ex` (verified via `diff` → `IDENTICAL`), so behavior observed with that toolchain applies.
- Isolated-behavior claims (comprehension filtering, `OptionParser` inference, `String.to_unsafe_atom/1`) were verified by running snippets; the full task's file-deletion side effects were reasoned about against the surrounding modules (`deps.unlock.ex`, `project.ex`) rather than executed, since running it requires a full project fixture.

### Ruled out

- **`target: :target` in the `loaded_opts` comprehension (lines 44-46)** — `--target` is not in `@switches`, so `opts[:target]` is always `nil`. I verified that `value = opts[switch]` acts as a truthiness **filter** in an Elixir comprehension (ran it: `for {switch,key} <- [only: :env, target: :target], value = opts[switch] ...` yields only `[env: "dev"]` when `only` is set, `[]` when absent). The `:target` entry is dead/leftover, never producing a stray option. No behavioral effect.
- **`--unlock` forwards the raw `args` to `deps.unlock` (line 78)** — potential concern that `--build`/`--only dev` would break `deps.unlock`'s parse or that `dev` would be consumed as a bogus app. Verified with `OptionParser.parse` (non-strict, as `deps.unlock` uses): `--build` → `build: true` (ignored), and `--only dev` → `only: "dev"` (type inferred, `dev` **not** stolen into positional apps: `{[unlock: true, only: "dev"], ["foo"], []}`). No misbehavior.
- **Nested lock (`with_deps_lock` in `run/1` wraps a `deps.unlock` call that itself calls `with_deps_lock`)** — not a deadlock: `Mix.Project.with_build_lock`/`with_deps_lock` delegate to `Mix.Sync.Lock.with_lock`, documented as re-entrant ("This function can also be called if this process already has the lock. In such case the function is executed immediately." — `project.ex:993-994`).
- **String/atom type mismatch across branches** — all three `apps_to_clean` sources are strings (`checked_deps` returns `Path.basename` strings; `OptionParser` apps are strings), and `clean_source`'s `local` list is built with `Atom.to_string`, so `app in local` compares string-to-string. Consistent.
- **`File.rm_rf` error handling (`maybe_warn_failed_file_deletion`)** — `with {:error, reason, file} <- result` correctly matches the 3-tuple error shape and the `{:ok, _}` success falls through harmlessly (return value discarded by `Enum.map`/`each`). Correct.
- **Unbounded atom creation via `String.to_unsafe_atom` (line 46)** — input is the operator-controlled `--only` CLI value, exactly one atom per invocation; not attacker-driven network input and not exhaustible in a run. By-design, matching `deps.unlock`.
- **`clean_source` skipping local (`:path`) deps not present in `loaded_deps` under `--only`** — a path dep filtered out of the converge would miss the `local` guard, but path deps live at their `:path` location, not under `deps_path`, so `File.rm_rf(deps_path/app)` is a no-op on a nonexistent directory. No data loss.

