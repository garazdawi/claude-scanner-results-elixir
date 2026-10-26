# Threat Model: `elixir/lib/iex` (IEx — Interactive Elixir Shell)

## Protocol / standard implemented
This is **IEx**, Elixir's interactive REPL — not a wire protocol. There are no RFC/NIST specs; behaviour is governed by the **Erlang/OTP `io_protocol`** (group-leader `{:io_request, ...}` / `{:io_reply, ...}` messages), **OTP distribution** (for `remsh`/remote shells, authenticated by the Erlang **magic cookie**), and OTP's `:user_drv`/`:shell`/`:proc_lib` conventions. It is fundamentally a **code-execution engine**: its documented purpose is to evaluate arbitrary caller-supplied Elixir.

Documented public API entry points (`.ex`, not `.erl`):
- `IEx` — `IEx.pry/0`, `IEx.break!/*`, `configure/1`, `started?/0`.
- `IEx.Server.run/1` — starts a session (opts: `:prefix`, `:env`, `:binding`, `:dot_iex`, `:on_eof`, `:register`).
- `IEx.Pry` — `pry/2`, `break/*`, `annotate_quoted/3`, `whereami/3` (breakpoint/debug low-level API).
- `IEx.Broker` — `take_over/3`, `register/1`, `respond/4` (session takeover coordination).
- `IEx.Helpers` — shell helpers (`c/1`, `r/1`, `recompile`, `open`, `import_file`).

## Trust / contract boundaries
- **Evaluator input is fully trusted-by-design.** Anyone who can type at the prompt (or feed the group leader) gets arbitrary code execution as the BEAM node's user. IEx is *not* a sandbox; there is no injection boundary to defend inside `evaluator.ex`. The real boundary is *who can reach the group leader / distribution port*.
- **`:dot_iex` / `import_file`** — reads and **evaluates** a file from disk. An attacker who controls `.iex.exs` in CWD or `~` achieves code execution on shell start. Path is operator/local-user controlled.
- **`IEx.Broker` messages** are BEAM-internal (`{:take_over, pid, ref, location, whereami, opts}`) but `opts` carries `:evaluator` pid and `:binding`; `location`/`whereami` are formatted into output. Any co-node/co-tenant process that can send these participates in takeover.
- **`remsh` / `IEx.Broker.servers/1`** uses `:erpc.call(node, IEx.Broker, :shell, [])` across `Node.list()` — trust is the **distribution cluster**: any node sharing the cookie is fully trusted and can drive/observe shells.
- **`:user_drv.whereis_group` + `Process.info(pid, :dictionary)`** — `broker.ex` reads another process's dictionary to find `:evaluator`/`:shell`; assumes those keys are set by trusted IEx code.

## Attacker model
- **Primary defended boundary: network/cluster reachability of the node**, not shell input. If distribution is exposed and the cookie is weak/known, a remote peer is effectively an operator — this is the classic "distributed Erlang = RCE" exposure, out of IEx's control but the reason IEx is dangerous to expose.
- **In scope for per-file review:** crashes/hangs from *malformed control messages and IO replies* (broker takeover races, unexpected `:io_reply` shapes — see `wait_input/4`'s `unknown IO message` fallback), robustness of parsing (`__parse__/3` catches `kind, error`), and pry-consent bypass (`take_over?` prompting `Allow? [Yn]`).
- **Out of scope:** "user runs malicious code in the shell" — that is the feature.
- **Co-tenant BEAM process:** can read pids via `Process.info(:dictionary)` and send forged `:take_over`/`:continue`/`:respawn` messages to a server pid.

## Common attack patterns for this family
- **Distribution cookie / exposed EPMD → RCE** (the dominant real-world IEx/Erlang incident class).
- **Auto-loaded config file execution** (`.iex.exs`, akin to `.bashrc`/`.psqlrc` hijack).
- **Pry-consent bypass / takeover race** — two clients racing a takeover; `:already_accepted` handling in `broker.ex handle_call({:accept,...})`.
- **Group-leader / IO-protocol confusion** — forged `:io_reply` or a hostile group leader feeding the reader; note `run_without_registration` does `Process.link(group_leader())`.
- **Info-leak via inspection helpers** (`i/1`, `IEx.Info`) exposing internals.

## Supervisor / process structure
`IEx.App` starts `IEx.Supervisor` (`:one_for_one`) with three children: **`IEx.Config`** (settings/colors/prompt), **`IEx.Broker`** (GenServer; session registry + takeover state; **single point** — if it crashes, all pending takeovers are lost, sessions keep running), **`IEx.Pry`** (GenServer; breakpoint table `@table IEx.Pry`, recompilation). The **server** (`IEx.Server.loop`) and **evaluator** (`IEx.Evaluator`, a `:proc_lib` process) are *not* supervised children — the server `trap_exit`s and monitors the evaluator, respawning it on death (`rerun/5`). Killing the evaluator (`Process.exit(:kill)` on `^G` interrupt) is a normal, expected path — evaluator crashes must never propagate to Broker/Pry.

## Cross-file invariants
- **`server.ex` owns the evaluator lifecycle**; Broker only holds pids. Broker never assumes an evaluator is alive — it monitors and `refuse/4`s takeovers on `:DOWN`. A reviewer ruling out use-after-death in Broker should confirm the `handle_info({:DOWN,...})` path clears both `servers` and `takeovers`.
- **`pry.ex` prunes `@elixir_internals` from stacktraces** before they reach `broker.take_over`; `server.ex` interpolates `location`/`whereami` into `IO.puts` — neither sanitizes for terminal escapes, but content originates from `Macro.Env`/source lines (local trust).
- **`__parse__/3` (server) delegates to `IEx.Config.parser()`** and wraps all parser failures in `{:error, kind, error, stacktrace}`; `evaluator.ex parse/3` similarly funnels `TokenMissingError` to `:incomplete`. Reviewers should assume parser exceptions are *contained here* and never crash the server loop.
- **Consent gating for pry lives in `server.ex take_over?/6`** (`Allow? [Yn]`), not in Broker; Broker's `respond/4` trusts the boolean it is given. Do not assume Broker enforces authorization.
- **`register: false` sessions** (shells) are discovered dynamically via `:user_drv`/`erpc`, not via the `servers` map — takeover fan-out covers both sets in `Broker.servers/1`.