# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.elixir.ex
<!-- sha256: d5e29d8ec2adaaef745a79f76cbd6b5d8c3bc1d052ebcf4bf9f8da522112ba7e -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.elixir.ex
<!-- sha256: d5e29d8ec2adaaef745a79f76cbd6b5d8c3bc1d052ebcf4bf9f8da522112ba7e -->

_Re-reviewed on 2026-08-27 (second pass, hedge resolution)_

## Summary

`Mix.Tasks.Compile.Elixir` is thin glue that parses the `mix compile` command-line options and delegates all real work (file I/O, extraction, manifest handling) to `Mix.Compilers.Elixir`. No sink in this file crosses a dependency/network/archive trust boundary. **No security findings.**

The first-pass report carried three hedges, all in the "Risks / caveats" tail:
1. the `--tracer` → `Module.concat` trust-boundary claim "rests on reading `compile.all.ex` rather than an end-to-end `mix` run";
2. `Mix.Dep.cached/0`, `Mix.Project.config/0`, `Code.get_compiler_option/1` internals were "not traced";
3. `Mix.Compilers.Elixir` is "out of scope for this single-file review".

Resolution below. Hedge (1) is now **verified** by reading the full deterministic dispatch chain (a provenance claim — reading the forwarding code is the correct verification, and each hop forwards `args` verbatim). Hedge (2) collapses to operator/dependency-controlled config that is out of scope per the threat model; I confirmed the one sub-path that could plausibly create atoms from strings (`Code.get_compiler_option(:tracers)`) does not. Hedge (3) is a legitimate file-scope boundary, not an unverified assumption about *this* file — restated as scope, not a finding.

## Security

No findings.

### Ruled out

- **`Module.concat([&1])` on `--tracer` values (`pop_tracers/1`, line 258).**
  Genuine atom creation from a string — re-reproduced in the sandbox: `Module.concat(["Totally.Fresh...#{n}"])` raised `atom_count` 18503→18505 and returned a fresh `:"Elixir.Totally.Fresh..."` atom. The result is used as a compiler-tracer module reference (`tracers_opts/1`). **Provenance now verified end-to-end:** the string can only come from `OptionParser.parse(args, …)` at line 126; `args` is delivered by `Mix.Task.run("compile.elixir", args)` inside `run_compiler/2` (`task.compiler.ex:299`), which forwards its `args` verbatim (only optionally prepending `--force`, `maybe_prepend_force/2`); that `args` is `Mix.Task.Compiler.run(compilers, args)` from `compile.all.ex:74-75`, whose `args` is `Mix.Tasks.Compile.All.run/1`'s own argv. No hop injects, rewrites, or sources a `--tracer` value from a dependency, git ref, archive, lockfile, or network peer. A malicious dependency cannot add `--tracer` to the top-level invocation; a dependency that *could* influence compile options already runs arbitrary code at build time (`deps.compile` compiles its source), so this grants no escalation. Atom creation is the intended semantics (`Module.safe_concat/1` would reject a not-yet-loaded tracer). Developer CLI input is explicitly out of scope per the threat model. Hardening (not required): `Module.safe_concat/1`.
  Axes examined: input provenance (full argv dispatch chain read hop-by-hop), atom-creation reality (executed), escalation-from-dependency. Not examined: none.
  Confidence: high (not a finding).

- **`case tuple_or_atom` / `Keyword.get(properties, …)` in `infer_signatures/1` (lines 203–214).**
  A malformed `application/0` return (non-keyword `properties`, or an `extra_applications`/`included_applications` entry that is neither a 2-tuple nor an atom) raises `case_clause`/`Keyword` errors. When compiling a dependency this runs against the dep's own `mix.exs`. Outcome is a self-inflicted build abort only — a malicious dep can fail its own build trivially — and no atom is created from a string, no privilege boundary crossed. `properties` comes from `project.application()` (the project module's own compiled `mix.exs` code, RCE-by-design per threat model), and `Mix.Dep.cached()` yields operator/dependency-resolved dep structs (out of scope: an actor controlling these already has build-time code execution).
  Axes examined: crash reachability (malformed config → raise), string→atom (none present), trust boundary (self-inflicted build abort). Not examined: none material — `Mix.Dep.cached/0` internals treated as operator/dep config per threat model, and no code path here turns its output into a string-derived atom (the `for` comprehension binds existing `app` atoms only).
  Confidence: high (not a finding).

- **`Application.put_env(:logger, :compile_time_application, app)` (`with_logger_app/2`, line 230).**
  Global application-env write, but `app = Keyword.fetch!(config, :app)` is the project's own `:app` atom from its already-compiled `mix.exs`, and the prior value is restored in the `after` block (line 233). No externally derived key/value, no cross-tenant poisoning.
  Axes examined: key/value provenance (operator config), restoration (after-block balanced). Not examined: none.
  Confidence: high (not a finding).

- **`Code.get_compiler_option(:tracers)` merge (`tracers_opts/1`, line 263).**
  Newly checked in this pass to close hedge (2). Returns whatever was set via `Code.put_compiler_option/2` — operator/project config, a list of already-resolved tracer *atoms*, not strings. It is concatenated into `opts[:tracers]`; no `String.to_atom`/`Module.concat` is applied to its contents. So even if an operator seeds it, no string→atom sink exists here.
  Axes examined: value type (atom list, not string), sink presence (no atom-from-string on this path). Not examined: none.
  Confidence: high (not a finding).

- **`srcs`/`elixirc_paths`, `compile_path`, `manifest()` paths.**
  All path components come from operator-controlled project config or the constant `@manifest` (`"compile.elixir"`); `Path.join` uses no external component and no `File` read/write/extract happens in this file. The `is_list(srcs) and Enum.all?(&is_binary/1)` guard (line 133) is reachable (set `:elixirc_paths` to a non-list) and correctly rejects malformed config — not dead code.
  Axes examined: path-component provenance (operator config / constant), guard reachability. Not examined: none.
  Confidence: high (not a finding).

- **`profile_opts/1` (lines 267–273).**
  Maps the `:profile` string to the literal atom `:time` (`{:ok, "time"}`) or deletes the key for any other value; never calls `String.to_atom` on the untrusted value. This is the safe pattern, not a sink.
  Axes examined: string→atom (only literal `:time` produced), fallthrough (deletes key). Not examined: none.
  Confidence: high (not a finding).

## Risks / caveats (scope, not unverified assumptions)

- The actual compilation, manifest parsing, `.beam` writing, `@external_resource` stat-ing, and any path handling live in `Mix.Compilers.Elixir` (`lib/mix/lib/mix/compilers/elixir.ex`), delegated to at line 166. Those file-format and path sinks belong to that file's own review; they are out of scope for *this* file, not an unverified claim about *this* file. Per the cross-file invariant, a reviewer of `elixir.ex` must not assume its inputs are integrity-verified here.
- `Mix.Dep.cached/0` and `Mix.Project.config/0` outputs are treated as operator/dependency-controlled config, consistent with the folder threat model (an actor controlling these already has build-time code execution). I confirmed no path in *this* file converts their outputs into string-derived atoms or into file-write/extract sinks.

## Did I verify this?

- **`Module.concat/1` creates fresh atoms:** yes — executed in sandbox (`atom_count` +2, fresh atom returned).
- **`--tracer` provenance is developer CLI only:** yes — read the full dispatch chain `compile.all.ex:74-75` → `task.compiler.ex:278-301` (`run/2`, `run_compiler/2`, `maybe_prepend_force/2`) → `Mix.Task.run("compile.elixir", args)`, confirming `args` is forwarded verbatim with no dependency/network injection point. This is a provenance claim; reading the forwarding code is its verification.
- **`Code.get_compiler_option(:tracers)` is not a string→atom sink:** verified by reading `tracers_opts/1` — its contents are merged, never `to_atom`-ed.
- **`Mix.Compilers.Elixir` internals:** not verified here by design (separate-file scope); flagged as a caveat, not relied upon for any ruled-out judgment about this file.
- **Did NOT run** a full `mix compile` end-to-end; the dispatch is deterministic and every hop was read directly, which is sufficient for the provenance judgment.

## Dropped in re-review

None. All first-pass ruled-out entries are preserved; hedge (1) is upgraded to `Not examined: none` (verified) via the executed atom-creation check plus the fully-read forwarding chain, and hedge (2)'s one plausibly-live sub-path (`Code.get_compiler_option(:tracers)`) was checked and added as an explicit ruled-out entry. No assumption required upgrading to a low-confidence finding.
