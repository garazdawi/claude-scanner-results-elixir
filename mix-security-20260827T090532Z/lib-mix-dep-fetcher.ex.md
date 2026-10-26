# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/fetcher.ex
<!-- sha256: be79646e6ebd1aee4301754ca4ebfe70bdf99350c3f6dd40d80936763553e3d8 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/fetcher.ex
<!-- sha256: be79646e6ebd1aee4301754ca4ebfe70bdf99350c3f6dd40d80936763553e3d8 -->

_Re-analysed on 2026-08-27_

The first pass ruled out the `mark_as_fetched/1` glob/`File.rm` traversal on the belief that the only source of a `../`-bearing `app` name was an **evaluated** `mix.exs` (i.e. RCE-equivalent). I read the two callgraph neighbours the hedge pointed at — `Mix.Dep.Loader` (`loader.ex`) and `Mix.Dep.Converger` (`converger.ex`) — and traced `app` back to its origins. One origin falsifies that belief: `Mix.Rebar.load_config/1` obtains dependency `app` atoms from **`:file.consult/1` of a transitive dependency's `rebar.config`** (`rebar.ex:79`), which is *data consulting, not code execution*. I executed the consult and the glob and confirmed both. That hedge is therefore upgraded to a low-confidence finding. The remaining hedges (cycle recursion, SCM dispatch, format-string, CLI atom-minting) were either verified against the neighbours or are genuinely resident in other files.

## Summary

One low-confidence finding: `mark_as_fetched/1` (lines 128–141) builds a `Path.wildcard` + `File.rm` deletion from an interpolated dependency `app` name, and `app` can carry `../` from a **non-executed** source — a transitive `rebar.config` read via `:file.consult` — giving an RCE-free, tightly-constrained file-deletion primitive (deletes files named exactly `.mix/compile.elixir_scm` at an attacker-traversed path). All other dangerous-looking sinks are fed by operator CLI arguments, are guarded upstream (cycles rejected by the converger's topological sort), or live in other files.

## Security

### Finding 1 — Path traversal in `mark_as_fetched/1` via non-executed dependency metadata (lines 128–141)

**Upgraded from hedged assumption in first pass.**

**Confidence: Low.**

`mark_as_fetched/1` interpolates the dependency `app` name into a glob and deletes every match:

```elixir
build_path
|> Path.join("*/lib/#{app}/.mix/compile.elixir_scm")
|> Path.wildcard(match_dot: true)
|> Enum.each(&File.rm/1)
```

`app` is always an atom (the loader's `with_scm_and_app` guards require `is_atom(app)` — `loader.ex:149,153,161`), but neither the loader nor the converger validates that atom against `../` or glob metacharacters. Nowhere in `converger.ex`/`loader.ex` is `app` checked for path safety; the only constraint is atom-ness.

The first pass asserted the sole taint source was an *evaluated* `mix.exs` (RCE). That is false. `Mix.Dep.Loader.rebar_dep/rebar_children` (`loader.ex:344–388`) calls `Mix.Rebar.load_config`, which does `:file.consult(config_path)` on a checked-out dependency's `rebar.config` (`rebar.ex:79`). `:file.consult` **creates atoms from the term data without executing it**. `Mix.Rebar.deps/parse_dep` then passes that atom straight through as the dependency `app` (`rebar.ex:124–162`). A malicious *transitive* Rebar/Erlang dependency — trusted by the operator only as an ordinary library — can declare `{deps, [{'../../../../../../tmp/evil', {git, "…"}}]}` and inject a traversal-bearing `app` with no code execution on the build host during `deps.get`.

Reachability into the sink: for such a new, unfetched dep, `do_fetch/3` hits the `out_of_date?` branch and calls `mark_as_fetched([dep])` at line 68 *before* any checkout; `scm.fetchable?()` is true for a git source, so the `for` comprehension's guard passes.

**Executed evidence:**
- `:file.consult` on `{deps, [{'../../../../../../tmp/evil_app', {git, "…"}}]}.` returned the atom `'../../../../../../tmp/evil_app'` (`is_atom` = true), confirming a non-executed source mints a traversal atom.
- Reconstructing `mark_as_fetched`'s pipeline with `app = :"../../../target"` and a populated `_build/dev/lib/` produced pattern `_build/*/lib/../../../target/.mix/compile.elixir_scm` and `filelib:wildcard` resolved it to `_build/dev/lib/../../../target/.mix/compile.elixir_scm`, i.e. the escaping path an operator's `_build` would `File.rm`.

- **Failure scenario:** Operator adds a trusted-looking Erlang/rebar git dependency; a compromised transitive `rebar.config` in that tree declares a dep whose app atom is `'../../../<path>'`. During `mix deps.get`, `mark_as_fetched/1` deletes `<path>/.mix/compile.elixir_scm` (relative to `_build/*/lib`) if such a file exists — with no code from the malicious dep having run.

- **Why Low, not higher:** the deletion primitive is severely constrained — the fixed suffix `/.mix/compile.elixir_scm` means only files with exactly that name (Mix SCM manifests) can be removed, not arbitrary files; the practical impact is forcing recompilation of a targeted Mix build tree. It also requires the operator to use the Rebar/Erlang dependency path and a `_build/*/lib` layout to exist. Confidence is Low because I confirmed the two mechanism halves in isolation but did not drive the full `deps.get` converger pipeline end-to-end (the assembled Mix build environment is not present in the sandbox).

- **Safe form:** reject or escape `app` names containing path separators / `..` / glob metacharacters at the loader boundary (where `is_atom(app)` is already enforced), or build the target path without shelling the app name through a wildcard.

### Ruled out

- **`String.to_unsafe_atom/1` atom creation (line 164, `to_app_names/1`; same call in `deps.get.ex:41`, `deps.update.ex:61,81`).** Genuine unconditional atom-minting sink (verified in first pass: `String.to_unsafe_atom("brand_new_atom_xyz_123")` produced the fresh atom). But `to_app_names/1`'s only inputs are the dependency names typed on the command line (`by_name`'s `names`) — operator-controlled, bounded by `ARG_MAX`, well below the atom-table limit; repeats don't accumulate. The attacker in this threat model is a remote source, not the invoking operator, so this is out of scope. (Note: bulk atom-exhaustion from a hostile `rebar.config` is real but is minted by `:file.consult` in `rebar.ex`, not by this line.)
  - *Axes examined:* input provenance (CLI `rest`/`names` only), unboundedness, threat-model actor. *Not examined:* none.

- **`Path.wildcard` + `File.rm` traversal (lines 133–138).** Not ruled out — see Finding 1.

- **`do_with_depending/2` unbounded recursion on a dependency cycle (lines 151–160).** Reads `all_deps` produced by `Mix.Dep.Converger.converge/4`. Verified against the neighbour: `converge/4` calls `topological_sort/1`, which builds a `:digraph` and, when `:digraph_utils.topsort` returns `false` (a cycle), raises `"…dependencies form a cycle…"` (`converger.ex:32–42`); it also raises on a self-dependency (`converger.ex:23–25`). So by the time `do_with_depending/2` runs on `all_deps` in `do_finalize`, the graph is acyclic and the recursion is bounded by the (finite) dep set. No non-RCE path produces an unbounded cycle here.
  - *Axes examined:* upstream cycle rejection (converger topsort), self-loop rejection. *Not examined:* none.

- **`scm.checkout(opts)` / `scm.update(opts)` dispatch (lines 71–77).** This file only dispatches to an already-resolved SCM module. The dangerous fetch logic (git argument/URL injection, redirect handling, `--sha512` checksum ordering) genuinely resides in `Mix.SCM.Git` / archive-hex installers, not here — confirmed the SCM module is selected upstream in `loader.ex:223–227` (`get_scm/2` → `Mix.SCM.available()`), so this file holds no fetch-transport code to review. Those sinks belong to the SCM files' own reviews.
  - *Axes examined:* presence of transport/injection code in this file (none), SCM resolution locus (loader). *Not examined:* the SCM modules' internals — out of this file's scope by construction, to be covered where they live.

- **`shell.error("* #{Mix.Dep.format_dep(dep)}")` / `format_status` (lines 174–177, `show_diverged!/1`).** `Mix.shell().error/1` emits a plain binary with no `~p`/`~s` format-string semantics, so this is not a format-string injection sink (verified in first pass that the argument is a pre-built binary, not a format template). Terminal-escape bytes in the interpolated status would still require a hostile evaluated `mix.exs` or a rebar-injected status string; the effect is at most cosmetic terminal-escape output (CWE-117), which is Info-level and not upgraded.
  - *Axes examined:* format-string semantics of `shell.error/1`, injection provenance. *Not examined:* none.

## Did I verify this?

- **Verified by execution (this pass):** (1) `:file.consult` mints a `../`-bearing atom from `rebar.config` data without executing it; (2) the `mark_as_fetched` glob pipeline (`Path.join` + `filelib:wildcard`) resolves an escaping traversal path from such an app name. Both runs shown under Finding 1.
- **Verified by reading the callgraph neighbours:** `loader.ex` (`with_scm_and_app` requires `is_atom(app)` but performs no path-safety check; `rebar_dep`/`rebar_children` feed rebar `app` atoms in); `converger.ex` (`topological_sort` rejects cycles and self-deps; no `app` path validation anywhere); `rebar.ex` (`load_config` → `:file.consult`; `parse_dep` passes the consulted atom through as `app`); `lock.ex` (lockfile is `Code.eval_quoted`'d — RCE-equivalent — but its keys do not construct the `Mix.Dep.app` used by `mark_as_fetched`, so it is not the traversal vector).
- **Not exercised end-to-end:** the full `mix deps.get`/`deps.update` converger run against an assembled Mix project — the surrounding build environment is not present in the sandbox — hence Finding 1 is rated Low confidence rather than Medium.

## Dropped in re-review

- **Dropped the dismissal "`Path.wildcard` + `File.rm` … the only way to get a traversal/glob-metachar app-name into the converged dep tree is to declare it in a `mix.exs` that Mix evaluates (RCE)."** New executed evidence: `Mix.Rebar.load_config` (`rebar.ex:79`) obtains `app` atoms via `:file.consult` — data consulting, not code execution — from a transitive dependency's `rebar.config`, and `:file.consult` mints `'../../../…'` atoms verbatim (run above). This falsifies the "RCE-only taint source" premise the dismissal rested on, so the entry is reclassified as Finding 1 (Low). The prior report's own caveat had flagged exactly this contingency ("if any such path exists that lets a non-executed source taint `app` … it should be re-rated"); the re-review confirms the path exists.
