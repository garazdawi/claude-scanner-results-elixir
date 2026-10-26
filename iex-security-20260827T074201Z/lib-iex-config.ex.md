# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/config.ex
<!-- sha256: 66b281a4f5f70cd47691179ae2cd5c8c0374529e485bf104c95392a343bae906 -->

_Analysed on 2026-08-27T07:50:43Z_

Analysis complete. I've traced every value that enters this file and confirmed the two behaviors that mattered: validation is fail-closed, and the ETS table is `:public` but only reachable by a co-tenant process (which already has code execution).

## Summary
`IEx.Config` is an internal configuration store whose only externally-influenced values (config options, prompt charlist) are operator/developer-controlled and correctly validated before storage; no untrusted-data path reaches a dangerous sink through a documented public API, so there are no security findings.

## Security

No findings

### Ruled out

- **`:public` ETS table stores executable `after_spawn` funs** (`handle_init/0`, line 171; `after_spawn/0`, line 161). Verified the table is `protection: :public, named_table: true` and holds raw funs (`after_spawn contents: [#Function<...>]`) that are later executed by the server/evaluator. A co-tenant BEAM process could `:ets.insert(IEx.Config, {:after_spawn, [fun]})` to inject a callback that runs when the next shell spawns. Excluded: writing to the table requires a process already running arbitrary code in the same VM (no trust boundary is crossed — that attacker already has RCE), the injection bypasses the module API rather than flowing through it, and the execution sink is in other files. `:protected` would be tighter (least privilege) but this is not reachable via a documented public API with untrusted data.
- **`parser/0` returns an MFA later dispatched via `apply` (line 74); `dot_iex/0` returns a path later read and evaluated (line 97).** Both are the classic IEx code-execution values, but within this file they are only `Application.get_env` reads. They are set exclusively by the developer via `IEx.configure/1` (validated: `tuple_size == 3` for parser, `is_binary` for dot_iex) or app config — operator-controlled, not forwarded from network/disk/dependency. The dangerous dispatch/evaluation lives in `evaluator.ex`/`server.ex`, not here.
- **`validate_option/1` / `handle_configure/2` (validation soundness).** Verified fail-closed: `Enum.each(options, &validate_option/1)` runs and raises `ArgumentError` at line 211 *before* any `Application.put_env` (stacktrace shows `validate_option` reached from `handle_configure:181`, the `Enum.each` line, ahead of `update_configuration`); a batch containing one bad option performs no partial write, and a previously-set valid value stayed intact (`width: 123`). Keys are constrained to `@keys`, so no atom injection and no key escapes into `Application.put_env`.
- **`Application.put_env` in `update_configuration/1` (line 192).** Values originate from the validated `configure/1` caller (documented entry point `IEx.configure/1`), which is the developer, not attacker-forwarded data. The `when key in @keys` clause guard is redundant given upstream validation (always true), i.e. harmless dead defensiveness, not a removed defense.
- **`prompt/1` `List.duplicate` (lines 29, 32).** Count is bounded by `max(0, ...)` (no negative) and by `:shell.prompt_width` of the configured, operator-controlled prompt; not attacker-influenced.
- **`inspect(option)` in the error path (line 211).** Error-message formatting of a developer-supplied config term; `inspect` is not an injection sink.
- **No atom-creation, `Code.eval*`, `System.cmd`/`:os.cmd`, `Path`/`File`, `Regex.compile`, `Base`/`URI`, crypto, or deserialization sinks are present in this file.**

## Risks / caveats
- The genuinely dangerous IEx surfaces — evaluating `.iex.exs`/`dot_iex`, applying the `:parser` MFA, running `after_spawn` funs, and the distribution/`remsh` trust boundary — are consumed in `server.ex`, `evaluator.ex`, and `broker.ex`, and should be judged there; this file only stores and returns those values.
- The `:public` ETS protection is broader than the read-from-any-process requirement needs (`:protected` would suffice and be least-privilege). It is not exploitable across a trust boundary because writing presupposes in-VM code execution, but it is a reasonable hardening note for a maintainer.
- An invalid option passed to `IEx.configure/1` crashes and restarts the `IEx.Config` agent (observed). This is a self-inflicted robustness quirk from the operator, not a security issue, and it occurs before any state mutation.

