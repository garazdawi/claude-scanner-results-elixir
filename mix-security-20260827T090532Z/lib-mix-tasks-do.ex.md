# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/do.ex
<!-- sha256: 43b94498e9a88eda36ad21247718e18740479fb26d53d1d91cb31eeebddcb265 -->

_Analysed on 2026-08-27T10:59:15Z_

I have completed the review. All sinks traced, the atom-creation path verified end-to-end through the public `run/1` API, and the `gather_commands/1` guard confirmed crash-safe on adversarial byte sequences.

## Summary

`Mix.Tasks.Do` splits `mix do` command-line arguments on `+` and dispatches each to `Mix.Task.run/2`; the only security-relevant sink is dynamic atom creation from `--app` values, whose sole input source is the operator's own command line.

## Security

#### `CSCAN-do-app-atom-creation`
- **Finding ID** — `CSCAN-do-app-atom-creation`
- **Severity** — Low
- **Confidence** — high
- **Location** — `do.ex:117`, `extract_apps_from_args/1` (reached from public `run/1` at `do.ex:72`)
- **Class** — Atom creation (CWE-770)
- **Impact** — Every distinct `--app <name>` value is passed to `String.to_unsafe_atom/1`, which interns a new atom that is never garbage-collected. This runs inside `extract_apps_from_args/1` *before* any task is dispatched, so the atom is created even if the subsequent task fails. In practice this is bounded and low-impact: the only input source is the operator's own command line (or a project's own `mix.exs` aliases) — no dependency, archive, git ref, network peer, or project-read file can steer the `--app` values of a `mix do` invocation (grep confirms the only callers are the Mix CLI dispatcher and tests). It therefore does not constitute a practical atom-exhaustion DoS, and it mirrors upstream Mix, which historically used the now-deprecated `String.to_atom/1` (this file uses the 1.21 non-deprecated replacement `String.to_unsafe_atom/1`; identical security semantics). I flag it because it is a live, unbounded dynamic-atom sink reachable from a documented public entry point, with a safe alternative available.
- **Evidence** — `verified`.
- `String.to_unsafe_atom/1` doc self-describes as unsafe: *"This function creates atoms dynamically and atoms are not garbage-collected. Therefore, `string` should not be an untrusted value…"* (`since: "1.21.0"`); `String.to_atom/1` is now `deprecated: "Use to_existing_atom/1 or to_unsafe_atom/1 instead"`.
- Mass creation proves no GC: `String.to_unsafe_atom("do_app_flood_#{i}")` over 1..5000 → `atoms before=14130 after=19130 delta=5000`.
- End-to-end via the public API:
  ```
  atom cscan_app_551 existed before? false
  run/1 raised (expected, after atom creation): Mix.Error
  atom cscan_app_551 exists AFTER run/1? true
  ```
  i.e. `Mix.Tasks.Do.run(["--app", uniq, "--app", "second_"<>uniq, "nonexistent_task_zzz"])` created `:cscan_app_551` (which did not previously exist) even though task dispatch subsequently failed.
- **Suggested fix** — The `--app` values only need to be compared against `Mix.Project.apps_paths/1` keys and forwarded to `run_in_apps/3`. Compare on strings against `known_apps |> Map.keys() |> Enum.map(&Atom.to_string/1)` (no atom creation), or use `String.to_existing_atom/1` and treat the `ArgumentError` as the "could not find application" warning case — either avoids permanently interning arbitrary operator-supplied strings.

### Ruled out
- **`gather_commands/1` comma guard (`do.ex:127-128`, `binary_part(head, byte_size(head), -1) == ","`).** A guard whose expression raises simply fails the clause rather than propagating, so `head = ""` (`binary_part("", 0, -1)` would raise) falls through harmlessly. Verified no crash on `""`, `","`, `"a,"`, invalid UTF-8 `<<0xFF, 0x2C>>`, multibyte-ending `"日本,"`, backslash, and `"+"`. No multibyte UTF-8 char ends in byte `0x2C`, so no false comma match on valid input. Input is operator CLI args regardless.
- **`Mix.Task.run(task, args)` / `Mix.Task.run_in_apps(task, apps, args)` (`do.ex:78-80`).** `task` is an operator-supplied CLI string; the atom/module resolution and dispatch happen in `Mix.Task` (other file), not here. Not a sink in this file, and input is operator-controlled.
- **`Mix.shell().info([:yellow, "warning: could not find application #{inspect(app)}"])` (`do.ex:106`).** `app` is an atom rendered via `inspect/1` into iodata; no format-string/`~s` log-injection sink, no untrusted formatting.

## Risks / caveats
- Under the folder's stated threat model ("Out of scope by design: the operator themselves"), the single finding above is arguably out of scope entirely — its only input is operator CLI input — and could reasonably be treated as informational/defense-in-depth rather than a vulnerability. I reported it as Low to document the live `to_unsafe_atom` sink and its safe replacement, not because it is practically exploitable.
- `Mix.Task.reenable/1`, `Mix.Project.config/0`, `Mix.Project.apps_paths/1`, `OptionParser.parse_head!/2`, and the actual task dispatch logic live in other files and were not reviewed here; I confirmed only that no untrusted-data path reaches this file's `run/1`.
- Reproductions were run against the sandbox toolchain (Elixir 1.21.0-dev / OTP 29). The `extract_apps_from_args/1` atom step was exercised through the public `run/1`; the private function itself was replicated only to characterize `to_unsafe_atom` vs `to_atom`, not relied upon for the reachability claim.

