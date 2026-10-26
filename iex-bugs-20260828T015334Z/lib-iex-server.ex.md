# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/server.ex
<!-- sha256: 7cec37e43c5ea065d2cefa40c756b3245ab4c81ac3d87797c08de653b3cceea2 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:13:48Z_

I've read the full file, tested key behavioral claims against the built `iex`, and confirmed the installed source is byte-identical to the file under review. Here is my report.

## Summary
The loop/takeover state machine and `__parse__` I/O-protocol handling are correct, but the `run_opts` typespec that `run/1`'s public `@spec` references omits the documented `:dot_iex` option, so it disagrees with both the `@doc` and the implementation.

## Bugfixes

#### `CSCAN-server-run-opts-missing-dot-iex`
- **Finding ID** — `CSCAN-server-run-opts-missing-dot-iex`
- **Severity** — Low
- **Confidence** — high
- **Location** — `server.ex:17-23` (`@type run_opts`), consumed by `@spec run(run_opts) :: :ok` at line 47; option used at `server.ex:364` and documented at `server.ex:41`.
- **Class** — Contract violation (`@spec`/`@doc` vs implementation mismatch)
- **Impact** — `run/1`'s docstring (line 41) documents `:dot_iex` as an accepted option, and `init_state/1` reads it (`Keyword.take(opts, [:dot_iex])`, line 364), but the `run_opts` type that `run/1`'s `@spec` is written against enumerates only `prefix`, `env`, `binding`, `on_eof`, `register` — not `dot_iex`. A caller who follows the published documentation and calls `IEx.Server.run(dot_iex: "path/to/.iex.exs")` passes a keyword whose key is not a member of the `run_opts` union type, so Dialyzer reports an invalid-argument type error even though the call works at runtime. The bad type then propagates into every analyzed caller of `run/1`.
- **Suggested fix** — Add `dot_iex: String.t()` to the `run_opts` type:
  ```elixir
  @type run_opts :: [
          prefix: String.t(),
          env: Macro.Env.t(),
          binding: keyword(),
          dot_iex: String.t(),
          on_eof: :stop_evaluator | :halt,
          register: boolean()
        ]
  ```
- **Threat note** — Not attacker-reachable; a pure spec/documentation-contract correctness defect affecting callers and static analysis of the documented public API.
- **Evidence** — `asserted`. By reading: the `@type run_opts` list (lines 17-23) contains no `dot_iex` key, while `@doc` line 41 documents `* :dot_iex - the path to the .iex.exs file to load` and `init_state/1` line 364 extracts it. The `@spec run(run_opts) :: :ok` (line 47) ties the type to the sole documented public entry point. (Runtime is unaffected — `run/1` handles the option correctly — so no execution trace applies; the defect is in the consumed type.)

### Ruled out
- **`counter` not incremented on error (`wait_eval`, line 193)** — Verified this is intended IEx behavior, not a bug: `printf 'raise "boom"\n1 + 1\n:done\n' | iex` shows the prompt staying `iex(1)>` after the error and only advancing to `iex(2)>` after the successful `1 + 1`. The `if(status == :ok, do: state.counter + 1, else: state.counter)` matches the shipped semantics.
- **`take_over?` arity dispatch (lines 79, 229, 237 vs defs 316/324)** — Checked each call site: the two auto-accept calls pass 4 args with a boolean `response` (matches `/4`, guarded `when is_boolean`), the prompt-then-respond call passes 6 args (matches `/6`). No misrouting.
- **Broker reply-once across the `:take_over` cond (lines 225-242)** — Each of the three cond outcomes issues at most one `IEx.Broker.respond` (branch 1 via `take_over?/4`, branch 2 via `take_over?/6`), and a refused/already-accepted result falls through to `callback.(state)` without a second respond, so the reply-once invariant holds on this side.
- **`__parse__/3` get_until continuation threading (lines 393-414)** — Traced the OTP `get_until` calling convention: initial continuation `[]` hits clause 2, which seeds the parser MFA; `{:incomplete, _}` returns `{:more, {counter, parser_state, mfa}}` whose 3-tuple continuation routes subsequent calls to clause 3; `:eof` and completion return `{:done, …, []}`. Parser exceptions in clause 3 are caught by its clause-level `catch` and surfaced as `{:error, kind, error, stacktrace}`, which `wait_input` (line 153) handles. Consistent.
- **Unknown/unmatched messages in `wait_*` loops** — `handle_common/6`'s catch-all (line 312) *receives* (consumes) the message before re-entering via `callback`, so unmatched messages are intentionally dropped rather than left to accumulate; not a mailbox leak.
- **`yes?/1` on non-string I/O results (line 320)** — `IO.gets` returning `:eof` or `{:error, _}` falls to `yes?(_) -> false` (line 344), correctly refusing the takeover rather than crashing.
- **Monitor/link handling in `run_without_registration`/`stop_evaluator`** — Re-linking the group leader and re-setting `trap_exit` on rerun are idempotent; `stop_evaluator` demonitors with `[:flush]` before signaling the evaluator. No ref leak on the observed paths.

## Risks / caveats
- The file is byte-for-byte identical to the installed Elixir 1.21.0-dev source (`diff` returned empty), i.e. pristine, heavily-tested upstream code; my behavioral checks used that build directly.
- Much of the takeover/pry surface (`{:take_over, …}`, `{:continue, …}`, `{:respawn, …}` messages) is an internal inter-process protocol driven by `IEx.Broker`/`IEx.Evaluator`, not the documented `run/1` API; per scope I did not treat crafted variants of those messages as in-scope, and I found no defect reachable purely through `run/1`'s documented options other than the typespec gap above.
- The single finding is a static-analysis/documentation contract defect with no runtime consequence, hence its `asserted` evidence basis (there is no execution to observe); the mismatch itself is directly readable in the source.

