# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/run.ex
<!-- sha256: 670e1885a85edbbf1c0ea2d511052fa7725cc5c20c8dca187fbfabc9c2eccfe4 -->

_Analysed on 2026-08-27T11:23:20Z_

Verification confirms: strict `OptionParser` rejects unknown flags (`atoms created by unknown flag: 0` — no CWE-770 via option names), and the file/argv branching matches the source. All parsed values trace to developer-typed CLI argv, and the two code-execution sinks are `mix run`'s documented purpose. Here is the report.

## Summary
`Mix.Tasks.Run` (`mix run`) is a developer-CLI task whose entire attack surface is operator-typed argv, and its two code-execution sinks (`Code.eval_string`, `Code.require_file`) are its documented, by-design purpose — no dependency/archive/network/git-ref/project-read data reaches any sink, so there are no security findings.

## Security

No findings.

### Ruled out

- **`Code.eval_string/1` on `--eval`/`-e` value** (`run.ex:106`, `run.ex:213-214` via `expr_evaluator`) — the value is developer-typed argv, and evaluating it is the documented feature ("evaluates the given code", moduledoc line 23–25). Equivalent to `mix run script.exs`. No trust boundary crossed. The `expr_evaluator` argument is hardcoded to `&Code.eval_string/1` at the public entry `run/1`; substituting a custom evaluator requires calling the `@doc false` 5-arity `run/5` directly, which is out-of-scope internal misuse (and needs a caller who can already run code).
- **`Code.require_file/1` on positional file and `--require` patterns** (`run.ex:106`, `run.ex:167`, `run.ex:200`, `run.ex:210`) — the script filename / glob is developer-typed argv; running it is the task's purpose. Not a boundary crossing.
- **TOCTOU between `File.regular?(file)` and `file_evaluator.(file)`** (`run.ex:166-167`) — `file` is the developer's own script path; `require_file` executes it as code regardless, and a local attacker able to swap the file mid-check already has arbitrary write in the dev's working directory. Self-inflicted; no dependency/network data involved.
- **`Path.wildcard(pattern)` resource cost** (`run.ex:222`, `filter_patterns/1`) — the glob (including `**`) comes from the developer's `--require` argument in a single-shot CLI invocation; caller-controlled cost is not a boundary crossing.
- **`System.argv(argv)` global-state write** (`run.ex:143`) — documented behavior (argv is exposed to the script via `System.argv/0`); the data is developer-typed argv, not externally influenced.
- **Deprecated `--config`/`-c` → `Mix.Tasks.Loadconfig.load_compile(value)`** (`process_config/1`, `run.ex:184`) — `value` is a developer-supplied config path; config files are operator-controlled (RCE by design per the folder threat model), not an attacker boundary.
- **Atom-table injection via option names** (`OptionParser.parse_head!`, `run.ex:83-104`) — verified: `strict:` rejects unknown flags and creates zero atoms for them (`elixir opt_test.exs` → `unknown-flag REJECTED …`, `atoms created by unknown flag: 0`). The only atom keys produced are the fixed allow-list.
- **Interpolation in `Mix.raise`/`IO.warn`** (`run.ex:169`, `run.ex:207`, `run.ex:124`, `run.ex:179`) — values land in plain exception/warning message strings (`#{file}`, `inspect(value)`); no downstream interpreter (shell/format/query) consumes them, so no injection sink.

## Risks / caveats
- The heavy machinery reached from this file lives elsewhere and is out of this file's scope: `Mix.Task.run("app.start", args)` (compiles/starts the project and its dependencies — where dependency code actually executes) and `Mix.Tasks.Loadconfig.load_compile/1`. Any untrusted-dependency-code concern belongs to those modules' reviews, not `run.ex`, which merely forwards a fixed task name / developer-supplied path.
- I verified option parsing and argv branching against the source-built Elixir 1.21.0-dev / OTP 29 toolchain. I did not execute a full `mix run` end-to-end inside a project (would only re-demonstrate the by-design eval/require behavior); the eval/require sinks' by-design nature is established directly from the moduledoc and the hardcoded evaluators at `run/1`.

