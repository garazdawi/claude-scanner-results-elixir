# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.clean.ex
<!-- sha256: 11d176c0580f1bf3b22e4a2da8b8fb0183f85eff6d200dc082250b05097af30e -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.clean.ex
<!-- sha256: 11d176c0580f1bf3b22e4a2da8b8fb0183f85eff6d200dc082250b05097af30e -->

_Re-analysed on 2026-08-27_

Re-review to discharge the "outside this file" hedges from the first pass. Every previously-hedged claim was either driven to executed evidence in the OTP 29 sandbox or resolved by in-file dataflow plus the confirmed `%Mix.Dep{}` struct contract in the callgraph neighbour `dep.ex`. No assumption remains unverified; no finding is dropped or added.

## Summary
`Mix.Tasks.Deps.Clean` is a manually-invoked destructive maintenance task. Every dangerous operation — dynamic atom creation (`String.to_unsafe_atom`, line 46), recursive deletion (`File.rm_rf`, lines 132/148), and glob expansion (`Path.wildcard`, lines 39/88/130) — is driven solely by developer-supplied CLI arguments or by Mix-managed, basenamed, symlink-safe directory paths. The one dependency-metadata input in this file (`Mix.Dep.Converger.converge/1` output) is consumed only through `Atom.to_string/1` and `scm.fetchable?/0`; it never flows into an atom-creation or path-deletion sink. No dependency-, archive-, or network-controlled byte sequence reaches a sink here.

## Security

No findings

### Ruled out

- **`String.to_unsafe_atom("#{value}")` (line 46) — atom-table exhaustion (CWE-770).**
  Always-flag sink, but its input is `opts[:only]` / `opts[:target]`, both from `OptionParser.parse!(args, strict: @switches)` — CLI flags typed by the developer running `mix deps.clean`, not attacker data. The `target: :target` arm is dead: `target` is absent from `@switches`, so strict parsing raises on `--target`, and `opts[:target]` is always `nil` (filtered by the `value = opts[switch]` comprehension guard).
  *Executed (OTP 29):* `OptionParser.parse!(["--target","x"], strict: switches)` → `RAISED: 1 error found!`; `--only prod` → `{[only: "prod"], []}`; no opts → `{[], []}`; the full `loaded_opts` comprehension over `--only prod` → `[env: :prod]`; `opts[:target]` → `nil`. Self-inflicted atom growth from one's own CLI is out of scope.
  **Confidence: High.**
  *Axes examined:* switch presence in `@switches`, strict-parse rejection of unknown flag, comprehension guard filtering `nil`. *Not examined: none* (each pushed through the real `OptionParser` + comprehension).

- **`File.rm_rf/1` in `clean_build`/`clean_source` (lines 132, 148) — path traversal / arbitrary deletion (CWE-22).**
  Deletion targets are `Path.join(deps_path, to_string(app))` (line 147) and `Path.join(build_path, to_string(app)) |> Path.wildcard()` (lines 128–130), always confined children of `deps/` and `_build/`. `app` is either a developer-typed CLI arg or a `Path.basename/1` of an existing directory from `checked_deps` (line 91) — basename cannot contain a separator, and `Path.wildcard` never yields `.`/`..`.
  *Executed (OTP 29):* `Path.wildcard("_build/dev/lib/*")` → `["_build/dev/lib/foo"]`, basenames `["foo"]` (no `.`/`..`); `File.rm_rf("_build/dev/lib/foo")` on a symlink → outside dir returned `{:ok, ["_build/dev/lib/foo"]}` while `/tmp/rmtest/outside/precious.txt` **survived** (`File.exists?` → `true`) — `rm_rf` unlinks symlinks instead of following them, so umbrella/path-dep symlinks or a TOCTOU symlink swap cannot delete source outside the build tree.
  **Confidence: High.**
  *Axes examined:* wildcard dotpath emission, basename separator content, symlink-follow-vs-unlink on directory symlink. *Not examined: none* (all three pushed through real `Path.wildcard`/`File.rm_rf`). Caveat: symlink non-follow is standard OTP `:file` semantics but platform-dependent in principle (verified only on Linux/OTP 29).

- **`Path.wildcard` cost from `--only` (line 39) / directory count (line 88) — ReDoS-style blowup.**
  The interpolated `*#{opts[:only]}/lib` pattern and the wildcard walks are bounded by the developer's own CLI value and the project's own on-disk dep count; no external party influences the pattern or the tree size.
  **Confidence: High.** *Axes examined:* pattern source (CLI), walk breadth source (local FS). *Not examined: none.*

- **`Mix.Task.run("deps.unlock", args)` (line 78).**
  Forwards the developer's own CLI `args` to another Mix task; no untrusted data introduced in this file.
  **Confidence: High.** *Axes examined:* argument provenance. *Not examined: none.*

- **Shell interpolation `shell.info("* Cleaning #{app}")` (line 125) / `Mix.shell().error(... #{dependency} ...)` (lines 102, 115).**
  `Mix.shell()` writes to stdout/stderr, not an OS shell — no command injection. The interpolated app/dep names originate from developer-declared dependency names / basenamed local dirs, not a network peer. (Note: `#{app}`/`#{dependency}` are Elixir string interpolation, not a `~s`/format-directive sink, so no format-string angle applies.)
  **Confidence: High.** *Axes examined:* sink type (Elixir shell vs OS shell), name provenance. *Not examined: none.*

- **`Mix.Dep.Converger.converge/1` and `scm.fetchable?()` (lines 48, 137) — behaviour/module dispatch and metadata-to-sink flow.** *(Upgraded from hedged assumption in first pass — the first pass deferred this to "outside this file"; now resolved.)*
  `converge/1` executes `mix.exs` (developer code, RCE-by-design, out of scope) and returns a list of `%Mix.Dep{}` structs. Confirmed via the callgraph neighbour `lib/mix/dep.ex:57–81`: `defstruct scm: nil, app: nil, …` with `@type t :: %__MODULE__{scm: Mix.SCM.t(), app: atom, …}` — `:app` is an atom and `:scm` is an SCM module. **Crucially, safety here does not even depend on that contract:** in this file the converge output (`loaded_deps`) is consumed only at line 56 (`filter_loaded` → `Atom.to_string(&1.app)`), line 97 (`Atom.to_string`), and line 137 (`Atom.to_string(app)` + `scm.fetchable?()`). `Atom.to_string/1` on an already-existing atom creates no new atom, and the `File.rm_rf` target `app` comes from `apps_to_clean` (CLI args or `checked_deps` basenames), **not** from converge. So no converge/SCM-derived byte sequence reaches the atom-creation or path-deletion sinks in this file.
  **Confidence: High** (in-file dataflow is exhaustive; struct shape corroborated by neighbour). *Axes examined:* every consumption site of `loaded_deps` in this file; whether converge output reaches line 46/132/148. *Not examined: none within this file* — the internal correctness of `converge`/`Mix.SCM` implementations is a separate file's review, but their output cannot reach a sink here regardless of shape.

## Risks / caveats
- The heavy lifting of loading dependency metadata lives in `Mix.Dep.Converger.converge/1`, `Mix.Project.*`, and the SCM modules. This file only consumes their structured output (`%Mix.Dep{app: atom, scm: module}`, confirmed in `dep.ex`) via `Atom.to_string`/`scm.fetchable?`; any injection would have to originate in those modules and, per the dataflow above, still could not reach a sink in *this* file. Their internal review belongs to their own files.
- Exploiting the `File.rm_rf` sinks would require an attacker who can already write into your `deps/`/`_build/` tree (same-uid local attacker) or run a dependency compile hook — both already grant equal-or-greater capability than `deps.clean` offers, so they are not an escalation and are out of the stated attacker model.
- The `File.rm_rf` symlink-unlink behaviour was executed only on Linux/OTP 29; it is standard OTP `:file` semantics but platform-dependent in principle.

## Did I verify this?
- **`--target` dead code / strict parse:** Verified — ran `OptionParser.parse!` with `--target` (raised), `--only prod`, and empty args, plus the real `loaded_opts` comprehension.
- **`File.rm_rf` symlink non-follow + `Path.wildcard` no dotpaths:** Verified — ran against a real symlink pointing outside the tree; outside target survived.
- **converge output shape / non-flow to sinks:** Verified — read `lib/mix/dep.ex:57–81` for the `%Mix.Dep{}` struct/typespec, and traced every use of `loaded_deps` in this file to confirm it reaches only `Atom.to_string`/`scm.fetchable?`, never a sink.
- **Byte-identical to upstream 1.21-dev reference:** Carried from first pass (not re-run this pass); does not affect any finding.

### Dropped in re-review
None. All prior ruled-out entries are retained; the previously-hedged converge/SCM caveat is upgraded to a fully-resolved ruled-out entry with in-file dataflow evidence and a neighbour-confirmed struct contract.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`Path.wildcard` glob cost from `--only` (line 39) / directory count (line 88) — ReDoS-style blowup.** The interpolated `*#{opts[:only]}/lib` pattern and the wildcard walks are bounded by the developer's own CLI value and the project's own on-disk dep count; no external party influences them.
