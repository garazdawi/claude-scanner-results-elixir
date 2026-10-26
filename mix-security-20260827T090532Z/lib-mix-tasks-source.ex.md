# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/source.ex
<!-- sha256: 846012e9e2a8492c5c649c36e56c2da7e3a6371d2ecb8937cbc845f32fe5fbe1 -->

_Analysed on 2026-08-27T11:25:12Z_

## Summary
`Mix.Tasks.Source` (`mix source`) is a developer-CLI introspection task whose entire input surface is the operator-typed `argv`; its one in-file "dangerous" call (`Code.string_to_quoted!`) is parse-only and every code-execution/command sink lives in `IEx.Introspection` (a different file), so no untrusted-data path reaches a sink here.

## Security

No findings

### Ruled out

- **`Code.string_to_quoted!(module)` (line 40) — arbitrary code execution.** On the always-flag list, but this function only *parses* text to AST; it does not evaluate. Verified: `Code.string_to_quoted!("System.halt(0)")` returns the quoted tuple and runs nothing. Additionally `module` is `argv` typed by the developer, not data from a dependency/network/archive. No eval, trusted source.

- **`Code.string_to_quoted!` — atom-table exhaustion (CWE-770).** Parsing does mint atoms for aliases/atoms in the expression, but the input is a single developer-typed argument in a single-shot CLI process (no accumulation across requests), and the guard (`first in ?A..?Z or first == ?:`) admits only alias/atom expressions. Self-inflicted and bounded; not a cross-boundary finding.

- **`Code.string_to_quoted!` raising `SyntaxError` on malformed input.** Verified `mix source "A B C"`-shaped input raises an uncaught `SyntaxError` instead of the friendly `Mix.raise`. Robustness/cosmetic only on developer input — not security.

- **`IEx.Introspection.open_location(file, line)` under `--open` (line 51) — editor command execution.** This is the sharpest sink reachable from this task's documented API (`--open` builds and runs `ELIXIR_EDITOR`/`EDITOR`), but the command construction lives in `IEx.Introspection.open_location`, **not this file**. The editor string is developer-controlled env; the only dependency-influenced input (`file`) is the target module's recorded source path, which an attacker can shape only by getting a malicious module into the load path — i.e. after installing a dependency, which already runs arbitrary code at compile time (mix.exs/macros). No boundary the threat model defends is crossed, and the sink is out-of-file.

- **`IEx.Introspection.decompose(ast, __ENV__)` (line 41) and `source_location/1` (line 48).** External file; `decompose` breaks apart `Mod.fun/arity` AST and resolves aliases against the compile-time `__ENV__`, it does not evaluate the expression. Not in scope for this file.

- **`Mix.shell().info("#{Path.relative_to_cwd(file)}:#{line}")` (line 56) — terminal-escape / injection via printed path.** `line` is an integer; `file` is a path string reaching only a print sink. `Mix.shell().info` performs no format-string interpretation. The only way `file` carries hostile bytes (e.g. ANSI escapes) is a malicious dependency's compiled source path — already compile-time RCE, out of scope — and the impact would be at most terminal-escape noise. No injectable interpreter downstream.

- **`Mix.raise` interpolations (lines 45, 53, 60, 65) of `module`/`reason`/`message`.** Verified the message is a plain string with no `~`/format interpretation and no shell; `Mix.raise` merely constructs a `Mix.Error`. No injection.

- **`loadpaths!/0` (lines 72–83).** Invokes `Mix.Task.run("loadpaths", args)` with a hard-coded flag list — no external component in the argv, no injection.

## Risks / caveats
- The genuinely powerful behavior triggered by this task (`--open` → spawn `ELIXIR_EDITOR`, and `decompose`/`source_location` resolution) executes in `IEx.Introspection`. I reviewed only `source.ex` as instructed; if `open_location` builds its editor command via a shell without escaping `file`, that would be a real (though dependency-gated, since a hostile source path presupposes a malicious dep = compile-time RCE) command-injection — but it is not in this file and should be assessed where `IEx.Introspection.open_location/2` is defined.
- I did not exercise the full `mix source --open` path end-to-end against a compiled project (no editor configured in the sandbox); the parse-only, guard, and interpolation claims above were run directly and are `verified`.

