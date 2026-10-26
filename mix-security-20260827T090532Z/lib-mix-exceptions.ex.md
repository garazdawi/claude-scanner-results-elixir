# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/exceptions.ex
<!-- sha256: 7476faa6241c7e339f0491169060f3f1ba4b4ac267b33ecda162951129b86afc -->

_Analysed on 2026-08-27T09:42:17Z_

The toolchain confirms my analysis. Key results:

- `String.jaro_distance` cost is ~linear in the target length (1 MB → 220 ms/call; 300 task modules × 1 MB target → **24 s** of CPU). So a large task name *would* amplify into tens of seconds when constructing `Mix.NoTaskError`.
- An atom `task` raises `FunctionClauseError` inside the reducer.

Both hinge on the trust boundary. Tracing every raiser of `Mix.NoTaskError` (`Mix.Task.get!` at `task.ex:352`, `Mix.CLI.ensure_no_slashes` at `cli.ex:162`) shows `task` is always a developer-supplied task name (argv or a `mix.exs` alias target) — never data from a dependency payload, archive, git ref, network peer, or a project-read file. That places both behaviours outside this repo's threat model.

## Summary
This file only defines Mix's exception structs and formats their human-readable messages; every interpolated value (`task`, `target`, `expected`, `actual`, module names) is developer-controlled build metadata rendered through `inspect`/plain concatenation into a displayed string, with no untrusted-input path to a code, atom, path, command, or injection sink.

## Security

No findings

### Ruled out
- **`String.jaro_distance` DoS in `did_you_mean/1` (exceptions.ex:39–51).** Constructing `Mix.NoTaskError` loads all task modules and computes a Jaro similarity of `task` against every task name. Measured cost is ~linear in `task` length (verified: 1 MB target ≈ 220 ms/call; 300 modules × 1 MB ≈ 24 000 ms). Real amplification, but not a finding: every raise site (`Mix.Task.get!`, `Mix.CLI.ensure_no_slashes`) is fed a task name that originates from argv or a `mix.exs` alias — developer-controlled, not from a dependency/archive/network/file boundary. Self-inflicted slowness only.
- **Atom `task` crashes the reducer (exceptions.ex:49).** `String.jaro_distance(source, target)` requires binaries; passing an atom task name raises `FunctionClauseError` (verified: `String.jaro_distance("compile.elixir", :foo)` → `FunctionClauseError`). Requires the caller to pass an atom (a caller-contract deviation), triggers only while already raising a not-found error, and the input is not untrusted — a robustness nit, not a security issue.
- **`inspect(task)` / `#{actual}` / `#{expected}` in message strings (lines 15, 20–21, 64, 78–79).** Values are only concatenated into an error message that is displayed; there is no downstream interpreter (no shell, EEx, SQL, regex, or `to_atom`). `inspect` escapes strings/control chars, so there is no log- or terminal-injection sink in this file. No format-string semantics exist in Elixir interpolation.
- **`expected_mod_name/1` → `Mix.Utils.command_to_module_name/1` (line 53–55).** Returns a *string* via `Enum.map_join` (confirmed at `utils.ex:675`), not an atom — no atom-table exhaustion. Reached only when a loaded task's name already equals `task` exactly.
- **`Mix.Task.load_all/0` side effect (line 41).** Loads (does not execute) `Mix.Tasks.*` beams already on the code path; placing such a beam already requires code/file access, and archive-as-plugin loading is by design per the folder threat model.
- **Compile-time `message =` in `Mix.NoProjectError` (lines 86–89).** A static literal; no file/network read at build time.

## Risks / caveats
- The "No findings" verdict rests on `task` (and the other exception fields) always being developer-controlled. If a downstream caller were to route genuinely untrusted, attacker-sized data into `raise Mix.NoTaskError, task: ...` — e.g. an application embedding `Mix.Task.run/2` on network input — the `did_you_mean/1` Jaro pass becomes a multi-second CPU amplifier and an atom value becomes a crash. Neither such caller exists in this tree; both would be defects in the *caller*, not this file.
- I did not audit `String.jaro_distance`'s own implementation source (it lives outside the partial checkout provided); I characterised it empirically against the built toolchain instead.

