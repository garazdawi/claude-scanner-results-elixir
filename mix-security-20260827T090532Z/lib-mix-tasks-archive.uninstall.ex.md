# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.uninstall.ex
<!-- sha256: bc99bd844f3934a22ff7c446de35293e965053a39a213082471958bbffc6a6e1 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.uninstall.ex
<!-- sha256: bc99bd844f3934a22ff7c446de35293e965053a39a213082471958bbffc6a6e1 -->

_Re-analysed on 2026-08-27_

## Summary

`Mix.Tasks.Archive.Uninstall` is a 28-line delegator. Its `run/1` forwards `argv` verbatim to `Mix.Local.Installer.uninstall/4`, where the only dangerous sink (`File.rm_rf!` on an operator-named archive path) resides. I re-read the delegate module (`installer.ex`) and swept the whole `elixir` tree for callers. Every path to the sink is fed by operator-supplied command-line `argv`; there is no untrusted-data-to-sink path within scope. **No findings.** The two first-pass hedges are now discharged with executed grep/read evidence (see "Did I verify this").

## Security

No findings

### Ruled out

- **Path traversal into `File.rm_rf!` via the archive name.** Confidence: high. `run/1` (`archive.uninstall.ex:26`) passes `argv` to `Mix.Local.Installer.uninstall/4`. I read the delegate: `name = List.first(argv)` (`installer.ex:304`) flows into `Path.join(root, name)` (`installer.ex:306-307`), and if that path exists it is handed straight to `File.rm_rf!(found)` (`installer.ex:314`). A name such as `../../outside` does escape the archives root — I re-confirmed by running `Path.join("/tmp/wctest/root", "../outside")` → `/tmp/wctest/root/../outside`, which `File.exists?` resolves as `true`, so the recursive delete would fire outside the archive dir. **Not a finding:** `name` originates solely from `argv`. The only entry point is `run/1`, and (see caller sweep below) every caller supplies a literal or shell-typed argv. The operator is explicitly out of scope (threat model: "the operator themselves"); an operator who can type `mix archive.uninstall ../../x` can equally type `rm -rf`. No dependency, archive entry, git ref, network peer, or project-read file feeds `argv`.
- **Glob injection via `Path.wildcard(name <> "-*")` in `matching_package/2`.** Confidence: high. Confirmed present at `installer.ex:333-335`: when the exact path does not exist, `name` (with glob metacharacters intact) is expanded by `Path.wildcard` and `List.first` of the matches is deleted. Same trust analysis — `name` is operator CLI input; self-inflicted, out of scope.
- **`MIX_ARCHIVES` controlling the deletion root.** Confidence: high. `Mix.path_for(:archives)` returns `System.get_env("MIX_ARCHIVES") || Path.join(Mix.Utils.mix_home(), "archives")` — verified at `mix.ex:657-659`. Operator-controlled environment, not attacker-influenced; out of scope.
- **Confirmation bypass via `--force`.** Confidence: high. `should_uninstall?/2` returns `opts[:force] || Mix.shell().yes?(...)` (`installer.ex:337-339`), so `--force` skips the prompt. Operator-supplied switch declared in this file's `@switches` (`archive.uninstall.ex:20-22`); documented behaviour, not a boundary crossing.
- **Compile-time / atom / eval / code-execution sinks.** Confidence: high. The file (re-read in full, 28 lines) contains only `use Mix.Task`, doc attributes, a static `@switches` literal, and the one-line `run/1`. No macro body, module attribute, or generator reads the filesystem/network or converts external bytes to atoms/terms/code. (Note the delegate's `parse_args` path does call `String.to_unsafe_atom` on package/app names — `installer.ex:255,259,286` — but that is on the *install* path, not reachable from `uninstall/4`, which this task is the entry to.)

## Risks / caveats

- The dangerous sink (`File.rm_rf!`), the wildcard matcher, and option parsing all live in `Mix.Local.Installer.uninstall/4` (`installer.ex:301-339`), shared with `escript.uninstall`. A full hardening review of that shared helper (e.g. confining deletions to the archives root via `Path.safe_relative/2` as defense-in-depth) belongs with `installer.ex`, not this delegator. I traced the flow enough to establish the name is operator-supplied; no cross-file finding results because the source is out of scope.

## Did I verify this

- **Hedge 1 (first pass): "did not exhaustively enumerate every programmatic `Mix.Task.run/2` caller."** RESOLVED. I grepped the entire `elixir` tree (case-insensitive) for `uninstall` and for `Archive.Uninstall`. The only invokers of `Mix.Tasks.Archive.Uninstall.run/1` are:
  - `test/mix/tasks/archive_test.exs:177,266,271,287,291` — all pass **literal** argv lists (`["archive-0.1.0", "--force"]`, `["archive"]`, etc.).
  - `lib/mix/hex.ex:69` — a **doc string only** (`"...uninstalling it with \"mix archive.uninstall hex.ez\""`), not a call.
  There is no internal caller anywhere in `lib/mix` that computes a `name` from dependency, archive, network, or project-file data and feeds it to this task. The argv source is exclusively the operator's shell command line. This is now verified by executed grep, not assumed.
- **Hedge 2 (first pass): trusted the delegate's data flow without reading it.** RESOLVED. I read `installer.ex` in full and confirmed line-by-line: `uninstall/4` at 301, `List.first(argv)` at 304, `Path.join(root, name)` at 306-307, `File.rm_rf!(found)` at 314, `matching_package` glob at 333-335, `should_uninstall?`/`--force` at 337-339. All prior claims match the actual code (line numbers now cited precisely).
- **Path.join traversal behaviour:** re-ran in the sandbox (`Path.join("/tmp/wctest/root", "../outside")` → `/tmp/wctest/root/../outside`, resolves outside root). Behaviour confirmed by execution; conclusion (out of scope, operator-only) unchanged.

## Dropped in re-review

None. All prior ruled-out entries are retained and upgraded from asserted to verified; no findings were dropped and no assumptions required upgrading to a low-confidence finding, because the caller sweep confirmed the argv source is operator-only.
