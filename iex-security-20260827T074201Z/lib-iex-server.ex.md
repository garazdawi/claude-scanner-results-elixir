# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/server.ex
<!-- sha256: 7cec37e43c5ea065d2cefa40c756b3245ab4c81ac3d87797c08de653b3cceea2 -->

# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/server.ex
<!-- sha256: 7cec37e43c5ea065d2cefa40c756b3245ab4c81ac3d87797c08de653b3cceea2 -->

_Re-analysed on 2026-08-27_

Both behavioral claims are confirmed by execution against Elixir 1.21.0-dev / OTP 29 built from the source under review. Every "outside this file" hedge from the first pass has been discharged by reading the named callgraph neighbour (`IEx.Evaluator`, `IEx.Broker`, `IEx.Pry`) — these are provenance claims ("where does this value come from / where does the sink live"), which a neighbour read legitimately settles.

**Confirmed (executed):**
- `inspect("\e[31mHACK\e[0m")` → `"\e[31mHACK\e[0m"` with the ESC escaped; `String.contains?(inspect(msg), "\e")` returns `false`. The `io_error("... #{inspect(msg)}")` / `inspect(detail)` paths cannot inject raw terminal control sequences.
- `yes?/1` fails **closed**: replicating lines 338–344, `:eof` → `false`, `{:error, :terminated}` → `false`, `"n"` → `false`, `""` → `true`. The consent gate accepts only the documented `[Yn]` default (empty/`y`).
- Over-limit atom literal (`":" <> String.duplicate("a",300)`) → `{:error, {_, "atom length must be less than system limit: ", ...}}`. The tokenizer refuses it rather than creating the atom; `__parse__`'s `catch kind, error` (line 411) would in any case convert a raise into `{:error, ...}`.

**Provenance discharged by neighbour reads:**
- `:dot_iex` read-and-evaluate sink is **not** in this file. `evaluator.ex:272–318`: `load_dot_iex`/`eval_dot_iex` do `File.read!(path)` + `:elixir.string_to_quoted!` + `eval_expr_by_expr`. `server.ex:364` only carries `:dot_iex` through via `Keyword.take(opts, [:dot_iex])` into `evaluator_options`. Confirmed the sink lives downstream.
- `take_location`/`take_whereami` provenance: `pry.ex:49–65` builds `location` from `Path.relative_to_cwd(file)` + `Exception.format_mfa` + line, and `whereami` from `whereami/3` source lines, then calls `IEx.Broker.take_over/3`; `broker.ex:101–111` forwards them verbatim into the `{:take_over, ...}` message. Both originate from `Macro.Env`/on-disk source (local trust), never from a network peer through a documented entry point.

## Summary
`IEx.Server` exposes no security defect reachable with untrusted data through its documented public API (`run/1`). Its only real inputs are caller-controlled session options and the trusted-by-design REPL byte stream; its parser-error and pry-consent paths are robust and fail-closed (both re-verified by execution). Every prior "outside this file" assumption is now confirmed against the neighbour modules.

## Security

No findings

### Ruled out

- **Terminal-escape injection via `IO.puts("Break reached: #{take_location}#{take_whereami}")` (lines 77, 227) and `io_error(... #{inspect(msg)/detail})` (lines 176, 306).** Confidence: High. `take_location`/`take_whereami` arrive only inside `{:take_over, ...}` messages; provenance now confirmed in `pry.ex:49–65` + `broker.ex:101–111` — they are built from `Path.relative_to_cwd`/`format_mfa`/source lines (local trust), not attacker-supplied. Reaching them still additionally requires sending a crafted message to this undocumented internal process (excluded by scope). The `inspect`-wrapped paths neutralize control bytes — executed: `inspect("\e[31m...")` renders `\e` escaped, no raw ESC. Axes examined: value provenance (via pry.ex/broker.ex read), inspect neutralization (executed). Not examined: none.
- **Pry-consent auto-accept — `take_over?(take_pid, take_ref, _, true)` at lines 79 and 229.** Confidence: High. The `true` (no-prompt) branch fires only for the shell's own evaluator (line 226 `evaluator == take_opts[:evaluator]`, i.e. self-pry) or during the `run_from_shell` startup window (line 79). Forcing it for a *foreign* evaluator requires forging a `:take_over` message with a matching `:evaluator` pid — a crafted message to an internal process, out of scope. The foreign-evaluator path correctly routes to the prompting `take_over?/6` (line 237). The prompt fails closed — executed: `yes?(:eof)`/`yes?({:error,_})`/`yes?("n")` → `false`. Broker's `respond/4` (`broker.ex:67–73`) trusts the boolean it is handed, so consent enforcement genuinely lives here, and here it is fail-closed. Axes examined: branch reachability, boolean fail-closed (executed), Broker trust contract (via broker.ex read). Not examined: none.
- **`code_fun.()` executed outside a try/catch (line 144).** Confidence: High. `code_fun` is always the `fn -> forms end` wrapper produced by `__parse__/3` (line 405). Substituting a raising/non-function value requires a hostile group leader (which already controls all IO and can crash the session regardless) — not reachable via `run/1`. Axes examined: value provenance (in-file, line 405). Not examined: none.
- **`start_evaluator/2` honoring `opts[:evaluator]` (line 112) to inject an arbitrary evaluator pid.** Confidence: High. `@doc false`, "made public but undocumented for testing"; the value flows from `run/1` opts (caller-controlled, trusted). Not an external boundary. Axes examined: source of `opts[:evaluator]`. Not examined: none.
- **`System.halt(0)` on EOF (line 149).** Confidence: High. Gated by `state.on_eof == :halt`, sourced from the caller-controlled `:on_eof` option (default `:stop_evaluator`); trusted. Axes examined: gate source. Not examined: none.
- **`apply(parser_module, parser_fun, args)` in `__parse__/3` (line 403).** Confidence: High. `parser_module`/`parser_fun` come from `IEx.Config.parser()` (operator configuration); `chars` is the trusted-by-design REPL stream; exceptions are contained by the `catch kind, error` clause (line 411 — executed: over-limit atom literal yields `{:error, ...}`, no crash). Axes examined: mfa provenance, exception containment (executed). Not examined: none.
- **Atom-table exhaustion via the parser path.** Confidence: High. Executed: a 300-char atom literal is refused by the tokenizer with "atom length must be less than system limit" rather than being interned; REPL input is trusted-by-design regardless. No `String.to_atom`/`binary_to_atom`/`Module.concat` exists in this file. Axes examined: tokenizer refusal of over-limit literal (executed), grep for atom sinks in-file. Not examined: none.
- **`:dot_iex` code execution on shell start.** Confidence: High. In this file `:dot_iex` is only carried through in `evaluator_options` (`server.ex:364`, `Keyword.take(opts, [:dot_iex])`). The read-and-evaluate sink is confirmed in `IEx.Evaluator` (`evaluator.ex:272–318`: `File.read!` + `:elixir.string_to_quoted!` + eval), not here. Axes examined: sink location (via evaluator.ex read), in-file data flow. Not examined: none.

## Risks / caveats
- The folder threat model flags forged `:take_over`/`:continue`/`:respawn`/`:io_reply` messages and takeover races as per-file concerns, but each requires a co-tenant BEAM process to send crafted messages to this undocumented internal server process — explicitly excluded by the review's trust model. If that co-tenant model is later brought in scope, the self-pry auto-accept (line 226) and the `Process.info(:dictionary)`-discoverable evaluator pid (`broker.ex:37–42`) deserve a second look.
- IEx is, by design, an arbitrary-code-execution engine; the meaningful boundary (distribution cookie / group-leader reachability) lives outside this file. Nothing here is or attempts to be a sandbox.

## Did I verify this?
- inspect ESC escaping: **verified (executed, OTP 29).**
- `yes?/1` fail-closed on `:eof`/`{:error,_}`/`"n"`: **verified (executed, replicated lines 338–344).**
- parser containment / over-limit atom refusal: **verified (executed, OTP 29).**
- `:dot_iex` sink is in `IEx.Evaluator` not `server.ex`: **verified (read `evaluator.ex:272–318` and `server.ex:364`).**
- `take_location`/`take_whereami` originate from `Macro.Env`/source lines: **verified (read `pry.ex:49–65`, `broker.ex:101–111`).**
- Broker `respond/4` trusts the consent boolean (enforcement is in this file): **verified (read `broker.ex:67–73`).**

## Dropped in re-review
None. All prior ruled-out entries are retained; each previously-hedged "outside this file" assumption was upgraded to `verified` by reading the cited neighbour module or by execution — no assumption needed to be downgraded to a finding.
