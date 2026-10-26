# /Users/lukas/code/claude-scanner/elixir/lib/mix/mix.exs
<!-- sha256: d032569d7714a3044dbbcbe9d7658972194f07f57a1afff34db1b50b8f1de5e9 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/mix.exs
<!-- sha256: d032569d7714a3044dbbcbe9d7658972194f07f57a1afff34db1b50b8f1de5e9 -->

_Re-analysed on 2026-08-27_

## Summary
`Mix.MixProject` is Mix's own static build-configuration file — two functions (`project/0`, `application/0`) each returning a hard-coded keyword list, plus a single `use Mix.Project`. No external input reaches any Phase 1 sink. Nothing exploitable here. The two hedges in the first pass ("`use Mix.Project` macro out of scope", "sinks live in other files") have both been resolved against the actual neighbour source and toolchain — see "Did I verify this".

## Security

No findings.

Walking every value in the file against the Phase 1 inventory:

- `version: System.version()` — the only function call. `System.version/0` returns a compile-time constant string baked into the Elixir release. Re-verified by execution: `elixir -e 'IO.inspect(System.version()); IO.inspect(is_binary(System.version()))'` → `"1.21.0-dev"` / `true`. Not influenced by any dependency, archive, git ref, network peer, or project file — not an external input.
- `app: :mix`, `escript: [main_module: Mix.CLI]`, `mod: {Mix, []}`, `registered: [Mix.State, Mix.TasksServer, Mix.ProjectStack]`, `env: [colors: []]`, `build_per_environment: false` — all compile-time literals (atoms and constants). No atom-from-bytes creation, no `Code.eval*`/`compile*`, no shell/`System.cmd`, no file/path op, no archive extraction, no `binary_to_term`, no network call, no regex, no crypto comparison, no shared-state write from external data.

No untrusted data crosses a trust boundary in this file, and there is no defensive check to reach — so nothing to report under "Dead / unreachable defensive code" either.

### Ruled out

- **`use Mix.Project` compile-time macro — verified.** Read `Mix.Project.__using__/1` at `lib/mix/lib/mix/project.ex:239-243`. Its parameter is `_` — the argument (here, none) is discarded entirely. The macro body is a fixed `quote` that only registers `@after_compile Mix.Project`. The after-compile callback `__after_compile__/2` (`:247-249`) calls `push(env.module, env.file)`, which pushes the compiled module and its own source file onto `Mix.ProjectStack`. No external/attacker-controlled input flows through the macro or its callback; the values pushed are the compiler's own `env.module`/`env.file`. No sink reachable from the `use` at this call site.
  - Axes examined: macro-argument handling (ignored via `_`), after-compile side effect (module/file push to project stack) | Not examined: none.
- **Compile-time execution of `mix.exs` — verified for this file.** `mix.exs` does run as code at build time (the folder's primary trust concern), but this specific file evaluates only `System.version()` (constant lookup, executed above) and constructs literal lists. It reads no file, hits no network, spawns no process, and evaluates no input. The "build machine is a target" concern has no fuel in this file.
  - Axes examined: every value in `project/0` and `application/0` (all literals except the executed `System.version()`), all call sites (the single `use`, resolved above) | Not examined: none.
- **`System.version()` as attacker input — verified.** Returns a compile-time constant (executed above). Anyone able to change it already controls the Elixir source tree/release being built — full build-host compromise, not a boundary this file defends.
  - Axes examined: return type (`is_binary` → true), value provenance (baked release constant) | Not examined: none.

## Risks / caveats
- The security-interesting Mix behaviour (archive extraction / `validate_archive_path!`, remote fetching via `Mix.Utils.read_path`, SCM `System.cmd("git", …)`, escript install, checksum verification) lives in other files under `elixir/lib/mix/`. This is not merely an out-of-scope assertion: this file contains no call — direct or via the `use Mix.Project` expansion (now read) — that reaches any of those sinks. They are correctly out of scope because this file does not invoke them, not because I declined to check.

## Did I verify this
- **`use Mix.Project` macro (was hedged "out of scope for this file"):** Verified by reading `lib/mix/lib/mix/project.ex:239-249`. The macro ignores its argument (`__using__(_)`) and only wires an after-compile hook that pushes the compiler's own module/file to the project stack — no external input, no sink. Hedge resolved; entry is now `verified`, `Not examined: none`.
- **"sinks live in other files, out of scope" (was a bare out-of-scope hedge):** Verified that this file makes exactly one function call (`System.version()`) and one macro invocation (`use Mix.Project`, read above); neither reaches the archive/network/SCM/escript sinks. The out-of-scope status is now established by tracing this file's outbound calls, not assumed.
- **`System.version()` constancy/type:** Re-verified by execution (`"1.21.0-dev"`, `is_binary` → true).

## Dropped in re-review
None. Both prior "Ruled out" entries are retained and upgraded from asserted to verified; no findings were dropped, and no hedged assumption needed upgrading to a low-confidence finding because each was cleared by executed/read evidence.
