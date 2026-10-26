# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.update.ex
<!-- sha256: d6600591c887a1c505416a24a2107c9113af2832a377030b62c793dd41971b40 -->

_Analysed on 2026-08-27T10:55:36Z_

I have completed the analysis. Let me summarize my findings.

**Trace of every sink in this file:**

The file is the `mix deps.update` task. Its only security-relevant sinks are three atom-creation sites via `String.to_unsafe_atom` (this codebase's repo-wide rename of `String.to_atom` — confirmed it unconditionally mints a new atom from arbitrary bytes):

- `deps.update.ex:61` — `{key, String.to_unsafe_atom("#{value}")}` on `--only` / `--target` values
- `deps.update.ex:81` — `String.to_unsafe_atom(app)` on positional dependency names

All three trace back to `run/1`'s `args`, which is the argv of `mix deps.update`. I confirmed there are **no internal callers** of `"deps.update"` — the task is CLI-only, so `args` is exclusively the operator's own command line. The lockfile read at line 68 (`Mix.Dep.Lock.read()`) is only *split by* the CLI-derived keys; its own keys are not atomized here. Modulo the `to_atom→to_unsafe_atom` rename, the file is byte-for-byte upstream Mix. I also verified the `value = opts[switch]` comprehension correctly filters nils (no spurious `:""` atom).

## Summary
`mix deps.update`'s only sinks are atom creations, and every one of them operates solely on the developer's own command-line arguments — no dependency, lockfile, archive, git ref, or network value reaches them — so there is no reachable security defect in this file.

## Security

No findings.

### Ruled out
- **Atom-table exhaustion via `String.to_unsafe_atom` (lines 61, 81)** — The values (`--only`/`--target` and positional dep names) come from `run/1`'s `args`, i.e. the argv the developer types for `mix deps.update`. No internal caller forwards attacker-derived args (grep for `"deps.update"` callers returned nothing), so no network/dependency/archive/lockfile data reaches these atoms. This is the operator attacking their own build — explicitly out of scope per the codebase severity model — and is identical to upstream Elixir's `String.to_atom`/`:"#{value}"`. The safe form (`String.to_existing_atom`) is *not* usable here because dep app names are legitimately new atoms during a fetch. Verified the primitive: `String.to_unsafe_atom("hello_world_test_atom_xyz")` → `:hello_world_test_atom_xyz`.
- **`Map.split(Mix.Dep.Lock.read(), to_app_names(rest))` (line 68)** — the potentially-untrusted `mix.lock` content is only used as the map being split; its keys are compared, not atomized, in this file. Any atom creation from lockfile keys happens inside `Mix.Dep.Lock` (out of scope).
- **`Mix.Task.run("archive.check", args)` (line 44)** — forwards argv to another Mix task; no shell/OS-command construction and no injection sink here.
- **`to_app_names` `else: app` fallback (line 81)** — benign type-normalization for programmatic callers passing atoms; for CLI callers `rest` is always binaries so the branch is effectively unused, but it guards nothing security-relevant, so it is not a "dead defensive code" finding.

## Risks / caveats
- I verified the atom-creation primitive and the comprehension's nil-filtering by running them (`elixir -e ...`), and traced input provenance by reading. I did **not** execute a full end-to-end `mix deps.update` (it requires a real project checkout plus network access), but the argv provenance of every sink is unambiguous from the source and the absence of internal callers.
- The genuinely untrusted-input atom/injection surface for the update flow lives in **other files** and is out of scope here: `Mix.Dep.Lock` (parses `mix.lock`, atomizes its keys), `Mix.Dep.Fetcher` / `Mix.RemoteConverger` (registry/network data), and `Mix.SCM.Git` (dep-supplied git refs/URLs → `System.cmd("git", …)`). A reviewer hunting atom exhaustion or command injection from a dependency should look there, not in this task.

