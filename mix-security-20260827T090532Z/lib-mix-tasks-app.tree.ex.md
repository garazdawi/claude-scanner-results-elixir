# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/app.tree.ex
<!-- sha256: 75364284d0a2612c0e68ac7f80d6ad18bcc3e5596644f1c9a14e1e74c4f8cb27 -->

_Analysed on 2026-08-27T10:22:47Z_

I have thoroughly traced every sink. Let me compose the final report.

## Summary

`Mix.Tasks.App.Tree.run/1` is a developer-run build task whose only externally-derived inputs are its own command-line arguments; the sole security-relevant sink is `String.to_unsafe_atom/1` interning atoms from those CLI strings, which is a weak-boundary (self-inflicted) atom-creation issue rather than an attacker-reachable one.

## Security

#### `CSCAN-app.tree-cli-arg-atom-creation`
- **Finding ID** — `CSCAN-app.tree-cli-arg-atom-creation`
- **Severity** — Low
- **Confidence** — low
- **Location** — `lib/mix/tasks/app.tree.ex:68` (`run/1`, `{opts, [app]}` clause) and `lib/mix/tasks/app.tree.ex:71` (`--exclude` mapping)
- **Class** — Atom creation (CWE-770)
- **Impact** — Both the positional application name and every `--exclude` value are passed straight to `String.to_unsafe_atom/1`, which interns a brand-new atom for any string (verified below). Atoms are never garbage-collected. The positional name is interned *before* the app-existence check, so even a name that corresponds to no application still permanently grows the atom table; and because `--exclude` is a `:keep` option, one invocation can intern arbitrarily many new atoms. A pathological invocation (on the order of ~10⁶ distinct `--exclude` values) could exhaust the atom table and abort the VM. **However**, the only source of these strings is the argument vector of `mix app.tree`, supplied by whoever runs the build. There is no path from a dependency, archive, git ref, network peer, or a project-read file to this argument list, and the interned atoms die with the short-lived build process. This is therefore self-DoS on the operator's own build shell, not an attacker-reachable sink — the behaviour is also identical to upstream Elixir, which uses `String.to_atom/1` at the same two sites. It is reported only to satisfy the "flag every unbounded atom-creation call and name the safe form" floor.
- **Evidence** — `verified`.
  - Sink creates atoms: `elixir -e 'String.to_unsafe_atom("cscan_totally_new_atom_xyz_12345")'` → `created atom: :cscan_totally_new_atom_xyz_12345`, `atoms added by to_unsafe_atom: 32`, and it equals `String.to_atom/1` on the same string.
  - End-to-end through the task: `mix app.tree this_app_atom_never_existed_98765` → `** (Mix) could not find application this_app_atom_never_existed_98765` — the nonexistent name is interned and echoed *before* the existence check fails.
  - `--exclude` path: `mix app.tree --exclude never_seen_exclude_atom_4242` parses and prints the tree, interning `:never_seen_exclude_atom_4242`.
  - These are genuinely new atoms: `String.to_existing_atom("brand_new_probe_9911_zzz")` raises `ArgumentError`.
- **Suggested fix** — Replace `String.to_unsafe_atom/1` with `String.to_existing_atom/1` at both sites and treat the resulting `ArgumentError` as the existing "could not find application" outcome (the atoms are used only for `Application.ensure_loaded/1`/`Application.spec/2` lookups and equality, so a not-already-interned name is by definition not a loadable app). Caveat: upstream deliberately uses the interning form to accommodate applications that are *available but not yet loaded* (whose name atom may not be interned yet); if that case must keep working, this is accepted upstream behaviour given the developer-controlled boundary and needs no change.

### Ruled out
- **`--output` file write / `.bak` rename (`write_dot_graph!` → `Mix.Utils.write_according_to_opts!`, utils.ex:399-404)** — the destination path is the CLI `--output` value (developer-controlled), the `.bak`-overwrite is explicitly documented, and the written content is inert DOT text. No attacker boundary; the TOCTOU between `File.exists?` and `File.rename!`/`File.write!` races only on the operator's own path. Cross-file anyway.
- **Printed `dot -Tpng #{inspect(file_spec)} ...` message (app.tree.ex:94)** — informational text only; nothing in this task executes `dot`, and both interpolated paths go through `inspect/1`. No command injection.
- **`Application.ensure_loaded/1` / `Application.spec/2` on the CLI-derived app (app.tree.ex:107,115-117)** — `ensure_loaded` loads only the `.app` resource, not modules, and does not run `start/2`; no code execution. Child app names from `.app` specs are already-interned atoms, so `children_for/2` creates no atoms.
- **`Mix.Task.run("compile", args)` (app.tree.ex:54)** — compiles the operator's own project with the operator's own args; not a boundary.

## Risks / caveats
- A non-security robustness bug exists but was not counted as a finding: two or more positional arguments (`mix app.tree a b`) match neither `case` clause and raise `CaseClauseError` at `app.tree.ex:57` (verified). Self-inflicted, no security impact.
- `Mix.Utils.write_dot_graph!`/`write_according_to_opts!` and the `String.to_unsafe_atom` implementation live in other files and were read only to the extent needed to judge this file's reachable behaviour; a full review of those belongs to their own files.
- I could not exercise a true umbrella/`@recursive` invocation, but the argument source (CLI) is unchanged in that mode, so the trust-boundary conclusion holds.

