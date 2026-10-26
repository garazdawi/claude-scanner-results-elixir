# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.loadpaths.ex
<!-- sha256: 902bfea4fc537995fd587ab607b5d325369c9ce6c580207fd7cc76120045341c -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.loadpaths.ex
<!-- sha256: 902bfea4fc537995fd587ab607b5d325369c9ce6c580207fd7cc76120045341c -->

_Re-reviewed on 2026-08-27_

## Summary

`Mix.Tasks.Deps.Loadpaths` is a thin orchestration task. Its only in-file inputs are operator-supplied command-line args (`args`) and the current top-level project's own `mix.exs` config (`Mix.Project.config()`). Every genuinely dangerous sink — archive checksum/extraction (`archive.check`), SCM fetch/git-ref handling, checksum verification, `_build` load-path construction (`Mix.Dep.load_paths`), and code compilation (`Mix.Tasks.Deps.Compile.compile`) — is delegated to other modules and is not fed data assembled *within* this file. No untrusted dependency/network/archive byte reaches an exploitable sink inside this file. **No findings.**

The re-review executed the two total-function claims from the first pass and read the four delegated-dispatch neighbours (`dep.ex`, `scm/git.ex`, `scm/path.ex`). One first-pass wording was corrected (`Version.parse_requirement/1` can raise, not only return `:error`), but the entry remains ruled out because its argument is operator-controlled `mix.exs` config.

## Security

No findings

### Ruled out

- **`Version.parse_requirement(req)` / `Mix.raise("... #{req} ...")` in `check_elixir_version/1` (lines 90–102).**\
  `req = config[:elixir]` is read from `Mix.Project.config()` — the *current top-level* project's `mix.exs`, which executes as code by design (operator/developer boundary, not a dependency's). **Correction to first pass:** I claimed `parse_requirement/1` "returns `:error` rather than crashing on malformed input." Executed evidence refines this: on a malformed *string* it returns `:error` (handled by the `:error ->` clause), but on a non-UTF-8 *binary* it raises `FunctionClauseError` (see run below). Both outcomes require the operator to have placed a bad value in their own `mix.exs`, so the crash is self-inflicted and does not cross a trust boundary. The `Mix.raise` interpolation of `req` goes to the shell as a plain message, not a format string. No code-execution or remote-DoS angle.\
  Confidence: High.\
  Axes examined: malformed-string input, empty-string input, non-UTF-8-binary input, boundary (who supplies `req`). | Not examined: none.

- **`inspect(dep.app)` / `reason` interpolation in `partition/4` (lines 151–153).** Fires only under `mix --debug`. `dep.app` is an atom rendered through `inspect` (control chars escaped). `reason = Mix.Dep.format_status(dep) |> String.split(", please run", parts: 2) |> hd()`. Executed evidence confirms `String.split/3` with `parts: 2` is total: `""` → `[""]`, no-match → `[whole]`, multi-match → 2-element list; `hd/1` therefore cannot crash. `Mix.Dep.format_status/1` is only reached here for statuses that pass `compilable?` (`:vsnlock`, `:noappfile`, `:scmlock`, `:depschanged`, `:compile`, `:envoutdated`) or `ok?` (`{:ok,_}`); reading `dep.ex:288–411` confirms every one of those statuses has a matching `format_status` clause, so no `FunctionClauseError` is reachable from this call site. Output goes to `Mix.shell().info/1`, which prints without re-interpreting format specifiers. No injection sink.\
  Confidence: High.\
  Axes examined: empty/no-match/multi-match split input, `hd` totality, reachable-status coverage of `format_status`. | Not examined: none.

- **`format_dep(dep)` / `format_status(dep)` printed via `shell.error` in `show_not_ok!/1` (lines 199–202).** These strings are produced in `Mix.Dep.format_status`/`format_dep` (`dep.ex:288–518`), not here. Reading those clauses confirms most tainted values (`vsn`, `path`, `req`) are rendered through `inspect`, but a few clauses interpolate raw — e.g. `:divergedreq` (`dep.ex:352`) emits `#{vsn}` and `#{app}` unescaped. The only residual angle is a terminal-escape sequence originating in dependency `.app`/version metadata reaching the developer's own console. That taint enters and is formatted in `dep.ex`, not in this file; within `deps.loadpaths.ex` only an already-formed string is printed on an error-abort path. Weak, non-boundary, and correctly attributable to `dep.ex`.\
  Confidence: High (attribution). The residual escape-sequence concern belongs to `dep.ex` `format_status` and is out of scope for this file.\
  Axes examined: which module builds the tainted string, inspect-vs-raw interpolation in the reachable clauses. | Not examined: none for this file (the escape-injection axis is a `dep.ex` concern, not reachable to a new sink here).

- **`dep.scm.fetchable?()` module dispatch in `local?/1` (line 188), and via `Mix.Dep.compilable?/ok?` (lines 147, 159).** `dep.scm` is resolved by Mix's converger to a registered SCM adapter module, not an attacker-supplied atom. Reading `scm/git.ex:10` and `scm/path.ex:10` confirms `fetchable?/0` is a nullary boolean predicate on each adapter (git → fetchable, path → not). `compilable?/1` and `ok?/1` (`dep.ex:470–500`) are pure pattern matches on the `status` field returning booleans. Not attacker-controlled module dispatch; no dynamic atom-to-module resolution occurs in this file.\
  Confidence: High.\
  Axes examined: source of `dep.scm`, return shape of `fetchable?/0` on both adapters, purity of `compilable?`/`ok?`. | Not examined: none.

- **`Code.prepend_paths(Enum.flat_map(all, &Mix.Dep.load_paths/1), cache: true)` (line 76).** The `_build/.../ebin` path strings are constructed inside `Mix.Dep.load_paths/1` (`dep.ex:520`), not in this file. No path component is assembled from external input at this call site; this file only flat-maps and forwards. Any path-traversal concern belongs to `load_paths/1` in `dep.ex`.\
  Confidence: High.\
  Axes examined: where the path strings are built vs. forwarded. | Not examined: none for this file (path construction axis lives in `dep.ex`).

- **`Mix.Task.run("archive.check", args)` (line 54) and `Mix.PubSub.start()/start_listeners()` (lines 51, 83).** `args` is the operator's command line. Archive checksum/extraction logic lives in `Mix.Tasks.Archive.Check`; listener-module execution lives in `Mix.PubSub`. Listener modules come from the documented `:listeners` config (trusted-dependency code). None of that data is assembled or validated in this file.\
  Confidence: High.\
  Axes examined: origin of `args`, location of the extraction/execution logic. | Not examined: none for this file.

- **PubSub-start-before-listeners ordering / `with_build_lock` double-check (lines 51, 106–123).** Deliberate concurrency *mitigations* (accumulate events to avoid missing concurrent compilations; grab-lock-then-recheck-then-compile). No exploitable check-then-act race for a remote or dependency attacker; the racing party is a co-resident local Mix process, already operator-trusted.\
  Confidence: High.\
  Axes examined: attacker model for the race (remote/dependency vs. co-resident local). | Not examined: none.

## Risks / caveats

- The security-relevant behaviour this task *triggers* — archive checksum/extraction (`archive.check`), SCM fetch and git-ref handling, dependency compilation (`Mix.Tasks.Deps.Compile.compile`), and `_build` load-path construction (`Mix.Dep.load_paths`) — all lives in other modules. Findings, if any, belong there. This is a scope statement, not a hedge: I read `dep.ex`, `scm/git.ex`, and `scm/path.ex` to confirm the delegated functions are pure/boolean where this file depends on them, and confirmed no path/command/eval/atom/deserialization sink operates on data that *enters within* `deps.loadpaths.ex`.
- The `:divergedreq` (and similar) `format_status` clauses in `dep.ex` interpolate dependency `vsn`/`app` without `inspect`, giving a latent terminal-escape (CWE-150/CWE-117) concern for dependency-controlled version strings reaching the developer console. That sink is in `dep.ex`, not this file; flagged here only to route it.

## Did I verify this?

- **`Version.parse_requirement/1` behaviour** — VERIFIED by execution (Elixir/OTP in sandbox): `"not a version"` → `:error`; `""` → `:error`; `<<0xff,0xfe>>` → raises `FunctionClauseError`. This corrected the first-pass wording (it does not uniformly "return `:error`"); the entry stays ruled out because the argument is operator `mix.exs` config.
- **`String.split(_, _, parts: 2)` / `hd` totality** — VERIFIED by execution: `""`→`[""]`, `"abc"`→`["abc"]`, multi→2-element list, `hd` on a match → binary. No crash path.
- **`format_status` clause coverage for statuses reachable from `partition/4`** — VERIFIED by reading `dep.ex:288–411`, `470–500`, `494–500`: every status admitted by `compilable?`/`ok?` has a `format_status` clause.
- **`fetchable?/0` return shape / non-attacker module dispatch** — VERIFIED by reading `scm/git.ex:10` and `scm/path.ex:10` (nullary boolean predicates) and confirming `dep.scm` is set by the converger, not from external input.
- **`load_paths`/`load_and_cache`/`check_lock`/`archive.check`/PubSub internals** — NOT independently re-verified end-to-end; deliberately out of scope because no data assembled in *this* file reaches those sinks. Confirmed the forwarding shape (flat-map/forward only) by reading the call sites.

## Dropped in re-review

None. Every first-pass ruled-out entry is preserved. One entry (`Version.parse_requirement`) had its supporting claim *corrected* (it can raise on non-UTF-8 binaries) but the dismissal itself stands on the operator-input boundary, now backed by executed evidence rather than the earlier over-broad "cannot crash" assertion.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`Mix.Task.run("archive.check", args)` and `Mix.PubSub.start_listeners()` (lines 54, 83).** `args` is the operator's command line; the archive-verification logic and the listener-module code execution live in `Mix.Tasks.Archive.Check` and `Mix.PubSub` respectively. Listener modules are the documented `:listeners` config mechanism (code from trusted deps), out of scope for this file.
