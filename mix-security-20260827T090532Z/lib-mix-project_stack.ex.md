# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/project_stack.ex
<!-- sha256: ff38977ff7ce1e6607d8b7fceb69dcfa2b130b466b387cc2fea29315a5f16151 -->

_Analysed on 2026-08-27T09:49:11Z_

Confirmed the edge behaviors I rely on. My analysis is complete. Here is the report.

## Summary
`Mix.ProjectStack` is an internal (`@moduledoc false`) GenServer that plumbs Mix's global project-stack and `post_config` state; its only file/path sinks operate on project paths and configs that originate *after* the corresponding `mix.exs`/config file has already been evaluated as code, so no untrusted input reaches a dangerous sink beyond the RCE Mix already grants by design.

## Security

No findings.

### Ruled out

- **`push/3` → `Path.join(Mix.Project.manifest_path(config), @manifest)`** — `config` and `file` reach this function only after `Mix.Project.push` has evaluated the project's `mix.exs` (the module is defined and `project/0` invoked). A malicious dependency already has arbitrary code execution at that point, so a path derived from its config is strictly weaker than the RCE it already holds. `@manifest` is a hard-coded constant (`"compile.lock"`), not attacker-influenced. No incremental traversal beyond by-design RCE.
- **`on_recursing_root/1` → `File.cd!(Path.dirname(file), fun)`** — `file` is a project's own `mix.exs` path already on the stack (post-evaluation). `fun` is an internal Mix callback, not external. `cd` into the project's own directory crosses no trust boundary the code execution hadn't already crossed.
- **`config_mtime/0` → `Mix.Utils.last_modified/1`** — operates over `config_files`, which are config-file *paths* stored via `push/3`/`loaded_config/2`. Only the mtime is read (not the contents), and the paths derive from already-executed project/config discovery. Reading a file's mtime is not a sensitive sink.
- **Stored/invoked funs (`callback.()`, `fun.()`, `after_compiler` via `prepend_after_compiler`/`pop_after_compiler`)** — all funs are supplied by internal Mix callers (`Mix.Project`, compiler tasks), not derived from external bytes. No `Code.eval`/`apply` on externally-derived text or module names.
- **Atom creation** — none. `name` is already a module atom, config keys/`:app`/`config_apps` are atoms from already-loaded code; no `String.to_atom`/`Module.concat`/`binary_to_atom` on external bytes.
- **Empty-stack `FunctionClauseError`/`MatchError` crashes** — `config_mtime/0`, `recur/1`, and both `update_stack` closures in `on_recursing_root/1` have only a `[h | t]` clause (and `on_recursing_root` additionally pattern-matches `[mid | bottom]` on the result of `Enum.split_while`, which returns `{whole, []}` when no entry is `recursing?` — verified above). These crash on an empty/non-recursing stack, but reaching them requires violating Mix's internal call-ordering invariants (these functions are only invoked while a project is on the stack); not reachable through any documented public API with attacker-shaped input. Self-DoS of the build at most, and only under internal misuse.
- **`config_mtime/0` read-then-write TOCTOU** — `get_stack` (compute-files branch) followed by a separate `update_stack` is not atomic; a concurrent pop to `[]` between them would crash the second call. The project stack is driven serially by the build; no attacker controls the interleaving. Robustness nit, not a security boundary.
- **Shared-mutable-state writes (`post_config`, `merge_config`, `loaded_config`, `compile_env`)** — this GenServer *is* the shared state, but every writer is Mix internal and every value originates from already-evaluated project code; one project poisoning another's stack entry is the documented behaviour of a project stack, not a cross-trust-boundary poisoning.

## Risks / caveats
- I reviewed only this file. `push/3`'s safety rests on the invariant — enforced in `Mix.Project` (not in scope here) — that a project's code is evaluated *before* its `config`/`file` are pushed. If any caller ever pushed attacker-controlled `config`/`file` *without* first executing that project's code, the `Path.join`/`File.cd!`/`last_modified` sinks would deserve re-examination; I could not rule that out from this file alone.
- The empty-stack crashes and the `config_mtime` TOCTOU are genuine latent robustness bugs (verified reproducible at the closure level), but I found no documented-public-API path that lets untrusted input drive them, so they are not reported as security findings.
- Nothing here performs network I/O, archive extraction, deserialization, atom creation from bytes, or shell/command execution, so those Phase-1 classes have no live sink in this file.

