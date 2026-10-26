# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/umbrella.ex
<!-- sha256: bbd35dac52c9d7493b282dc7214387ebe120be10b666feddda9a14c6142ce96e -->

_Analysed on 2026-08-27T09:40:12Z_

Verified: `File.ls` yields only real directory entries (never `.`/`..`), the atom↔string round-trip is faithful, and the resulting `Path.join`/`Path.expand` stay confined under the project root for normal names. Combined with my trace of `Mix.Project.apps_paths`, this settles the reachability question for every sink in the file. Writing the report.

## Summary

`Mix.Dep.Umbrella` only builds `%Mix.Dep{}` structs and build/dest paths from the umbrella project's own `apps_paths` (operator-controlled `:apps` config or `apps/` directory names) and delegates all loading to other modules, so it exposes no sink reachable by dependency-, archive-, git-, or network-supplied data.

## Security

No findings

### Ruled out

- **Path traversal via `dest_path = Path.expand(path)` / `Path.join([build, "lib", Atom.to_string(app)])` (lines 20–21).** Both `app` and `path` come from `Mix.Project.apps_paths/1`. The paths it returns are `Path.join(apps_path, <name>)` where `<name>` is either an operator-declared `:apps` atom or a `File.ls(apps_path)` directory entry, and each is only included after confirming a `mix.exs` regular file exists under it. Directory entries from `File.ls` can never be `.`/`..` or contain a separator, so they are single, non-traversing path components — verified: `File.ls("apps")` → `["my_app1","my_app2"]`, `contains . or ..? → false`, and `Path.expand("apps/my_app1","/proj") → "/proj/apps/my_app1"`. The only route that could inject `..` is the `:apps` list in `mix.exs`, which is operator-controlled and executes as code anyway (out of scope by the folder's attacker model).
- **Atom creation / CWE-770.** This file performs only `Atom.to_string(app)` (line 21), which consumes an existing atom and creates none. The atom-from-untrusted-bytes step (`String.to_unsafe_atom` on directory names) lives upstream in `Mix.Project.apps_paths` (`project.ex`), not here; verified the round-trip is faithful (`Atom.to_string(String.to_atom("my_app1")) == "my_app1"`).
- **Cache poisoning via `Mix.State.read_cache/write_cache` (line 52).** The key `{:umbrella_deps, Mix.env(), project}` and the cached `loaded()` value are derived entirely from trusted project state; there is no external component in the key or value. Co-resident mutation of Mix's ETS is the operator-trusted-local concern already noted in the folder threat model, not a remote-content boundary.
- **Loading child code in `loaded/0` (lines 67–76).** `Mix.Dep.Loader.load/3`, `validate_manifest/1`, and `Mix.Dep.Converger.topological_sort/1` do the actual config reading/dispatch; their logic is in other files. Here the results are only filtered to sibling umbrella apps (`dep.app in apps`), which narrows rather than widens trust.

## Risks / caveats

- **Upstream, not in this file:** `Mix.Project.apps_paths` (`lib/mix/lib/mix/project.ex:481–486`) converts every `apps/` directory name to an atom via `String.to_unsafe_atom/1` and accepts the `:apps` list without directory validation. That is the genuine atom-creation site and deserves review in `project.ex`; it is nonetheless gated behind the operator-trust boundary, since running any Mix task on a repository already executes its `mix.exs`.
- **Scope:** I reviewed only `umbrella.ex`. The behaviour of `Mix.Dep.Loader.load/validate_manifest`, `Mix.Dep.Converger.topological_sort`, `Mix.SCM.Path`, and `Mix.Project.build_path` was assumed correct and not audited.
- **Reproduction depth:** I verified the constituent facts (directory-listing safety, atom round-trip, path confinement) directly against the Elixir 1.21.0-dev / OTP 29 build under review, but did not stand up a full malicious-umbrella `mix compile` run; the file's data flow is short and fully traced, so this does not leave a reachable sink unexamined.

