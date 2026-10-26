# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/cli.ex
<!-- sha256: 33019d87162f21941309ff637745c8f16347625cf990337f58bb11cc9d15e8e3 -->

_Analysed on 2026-08-27T09:17:20Z_

## Summary
`Mix.CLI` is the `mix` command dispatcher; every sink it contains operates on operator-controlled input (CLI argv, environment variables, the project's own `mix.exs`, and the user's `~/.mix/config.exs`), and the genuinely attacker-controlled processing (dependency/archive fetching) lives in the tasks it routes to via `Mix.Task.run/2`, i.e. other files — so this file has no findings.

## Security

No findings

### Ruled out

- **`String.to_unsafe_atom(task)` — `maybe_change_env_and_target/2` (line 167).** This is unbounded atom creation (`to_unsafe_atom` demonstrably allocates a new atom — verified: `atom_count` rose by 32 for one new name, and the function is not exported). But `task` is the first CLI argument (`get_task/2` ← `System.argv()`), i.e. operator input, not data from a dependency/archive/network/git ref. Critically, each `mix` invocation is a fresh BEAM VM with a fresh atom table (verified `atom_count` starts small), so atoms cannot accumulate across invocations — there is no exhaustion vector. `to_unsafe_atom` is also a deliberate correctness choice here: task names in `preferred_envs`/`preferred_targets` keyword lists must be looked up even when the task's atom does not yet exist, so `to_existing_atom` would wrongly raise. A slash/newline-laden task name only yields a harmless quoted atom (verified), and validation (`ensure_no_slashes`) still runs before dispatch. Not exploitable across the folder's attacker boundary.
- **`Code.compile_file(mix_exs)` — `load_mix_exs/2` (line 68).** Arbitrary code execution, but `mix_exs` is the top-level project manifest (`MIX_EXS` env or `"mix.exs"`). Executing the project's own `mix.exs` is Mix's defining behaviour and is explicitly out of scope per the threat model ("`mix.exs`… run arbitrary code — by design, not a boundary"). Running `mix` in an untrusted cloned repo triggering this is the documented "install/build only from sources you trust" caveat, not a defect in this file. A *dependency's* `mix.exs` is compiled elsewhere (`Mix.Dep.*`), not here.
- **Missing `try/after` around `Code.put_compiler_option(:no_warn_undefined, …)` (lines 67–69).** If `compile_file/1` raises, `:no_warn_undefined` is left globally set to `:all`. Robustness only: an exception from compiling `mix.exs` aborts the whole `mix` run and the VM exits, so the leaked global compiler option is never observed. No security impact.
- **`Mix.Tasks.Loadconfig.load_compile(path)` — `load_dot_config/0` (line 264).** Compiles `~/.mix/config.exs` (or `MIX_HOME`/XDG location). Source is the local user's own config directory; writing there is already local privilege, not a remote/dependency boundary.
- **`MIX_PROFILE`/`MIX_PROFILE_FLAGS` parsing and `Mix.State.put(:profile, …)` (lines 28–38).** Inputs are environment variables (operator-controlled); `OptionParser.split/1` and `Tprof.parse!/1` operate on operator input, and the shared-state write derives from it. No cross-boundary source.
- **`run_task/2` exception handling (lines 133–143) and `simplified_exception/2` (line 147).** Clause ordering is sound (`{false, true}` is caught before the `code in 0..255` guard, and `true`/`false` never satisfy `in 0..255`); every branch is reachable and no defensive clause is dead. `simplified_exception`'s `%name{}` pattern matches all Elixir exceptions (structs). No injection sink beyond ordinary terminal output.

## Risks / caveats
- The checkout under review contains only `lib/mix` (no `lib/elixir`), so `String.to_unsafe_atom/1`, `Mix.Task.run/2`, `Mix.Project.*`, and `Mix.Utils.mix_config/0` were treated as external; I verified `to_unsafe_atom`'s atom-allocating behaviour against the installed Elixir 1.21.0-dev / OTP 29 build but could not read their source here.
- The security-relevant weight of `mix` invocation lies almost entirely in the *dispatched tasks* (`deps.get`, `archive.install`, `escript.install`, compilers) reached through `Mix.Task.run(name, args)`; those files — not this router — handle URLs, git refs, checksums, and archive entry names, and are where zip-slip/checksum/argument-injection findings would appear. This file forwards a validated task name and args and does not itself touch that untrusted data.
- I confirmed no caller inside the checkout invokes `Mix.CLI.main/2` with non-`argv` arguments; the real caller is the `mix` launcher (outside this checkout), which passes `System.argv()`. A hypothetical embedder calling `main/2` in a loop with hostile task names could leak atoms via `to_unsafe_atom`, but that is caller misuse of an internal (`@moduledoc false`) entry point with self-supplied input, outside the folder's attacker model.

