# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/pry.ex
<!-- sha256: 00514a57df6954a1b846fef2ee7212603f65caa3caf98463db775cb5c4cdf670 -->

_Analysed on 2026-08-27T08:01:04Z_

I now have enough to conclude. I traced every public entry point (`pry/2`, `whereami/3`, `break*`, `reset_break*`, `remove_breaks*`, `breaks/0`, `annotate_quoted/3`, `dbg/3`) and every sink the grep surfaced, and verified `whereami`'s robustness by running it against hostile-shaped inputs.

## Summary
`IEx.Pry` is a defensively-written breakpoint/pry engine whose sinks are all driven by the local developer's own modules, source, and macro-expansion AST; no untrusted network/disk/dependency/archive value reaches a dangerous sink through a documented public API, so there are no in-scope findings.

## Security

No findings

### Ruled out

- **`backend.debug_info/4` dynamic dispatch (line 496).** `backend` is read from the target module's own `{:debug_info_v1, backend, ...}` chunk (line 468). To reach it a developer must call `break/*` on an already-loaded module whose `.beam` carries a crafted debug backend. A module that is loaded has already had its code available to run; a malicious dependency shipping such a `.beam` executes when used regardless, and the standard Elixir backend is `:elixir_erl`. This is not untrusted data crossing a public-API boundary — it requires prior code execution, so it is out of scope.

- **`whereami/3` file read / DoS (lines 176–193).** Public and it does `File.stream! |> Enum.slice(min..max)`, but the path and line are chosen by the caller (the caller is not the attacker), the read is bounded by the source file, and the range is clamped by `line ± radius`. Verified robust: huge `line` → `:error` in 0 ms, missing file → `:error`, NUL-byte path → `:error`, `radius 0` → `FunctionClauseError` (guard). No crash or unbounded read reachable.

- **Terminal-escape passthrough in `whereami`/`location` (lines 52, 55, 199, 201).** `whereami_lines` returns raw source bytes unescaped — verified that `\e[31m` in the source survives into the returned chardata. However this file only *returns* chardata; the write-to-terminal sink is in `server.ex` (out of scope), and the content is source lines / `Macro.Env` file paths — to display attacker-controlled source you must already be executing that code. The folder threat model classifies this as local trust.

- **`String.to_unsafe_atom("arg" <> id)` (line 533).** Reached only via `instrument_clause` during `break`; `id` ranges `1..arity` with arity ≤ 255 for real functions, producing deterministic reused atoms `arg1..argN`. No atom-table exhaustion, no long/hostile atom string.

- **`:public` breakpoint ETS table (line 361) and `:ets.update_counter` in generated code (line 551).** A co-tenant BEAM process could poison/delete counter entries and crash an instrumented clause via `ArgumentError`. This is the standard `:public`-ETS co-tenant caveat (same class ruled out in sibling reviews), not reachable through a documented public API by remote/untrusted data.

- **Hard pattern match in `build_args_guard_condition` (lines 293–294) and in `instrument`/`deinstrument` (`{:module, _} = ...`).** These can `MatchError`, but only on `break/6` guard AST or module debug_info supplied by the developer at the prompt — self-inflicted, not an external boundary.

- **Compile-time `dbg/3` and `annotate_quoted/3` AST rewriting.** These run at macro-expansion time on the source being compiled, and only when the IEx `:dbg_callback` is active (an IEx session), not during ordinary `mix compile`/`mix deps.compile` of dependencies. No dependency-build sink.

## Risks / caveats
- The file's genuine security-relevant surfaces are *not here*: pry-consent gating lives in `server.ex` (`take_over?`), session takeover/authorization in `broker.ex`, and the terminal write of `location`/`whereami` in `server.ex`. A reviewer should not conclude "pry is safe" from this file alone — its safety depends on those callers, and on the node's distribution/cookie exposure (the dominant real-world IEx RCE class), which is outside any single file.
- The `:public` ETS table and the unescaped source bytes returned by `whereami` are deliberate design trade-offs under a local-trust model; they would become relevant if a future caller printed `whereami` output from a genuinely untrusted file path, or if the ETS table's integrity were ever relied upon for a security decision (it currently is not).
- I ran the `whereami` robustness checks against the built-from-source binary; the `backend.debug_info`/`break` recompilation path was reasoned about rather than executed end-to-end (it requires constructing a module with a crafted debug backend), but the trust analysis holds regardless of the runtime shape.

