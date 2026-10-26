# Correctness / Bug Profile — `elixir/lib/mix`

## What this code is
This is **Mix**, the Elixir build tool (part of the Elixir standard distribution, Apache-2.0). It is *not* a network protocol implementation — do not apply an RFC/wire-decoder threat model. It compiles projects, resolves and fetches dependencies, manages archives/escripts/releases, and runs tasks. No standards body governs it; the closest "specs" are its own conventions (the `mix.exs` project DSL, the `mix.lock` lockfile format, SemVer for versions, and the SCM contracts for Git/Hex/path dependencies).

**Public API entry points a caller/task author touches:** `Mix.CLI.main/2` (escript entry), `Mix.Task.run/2`, `Mix.Project` (config/stack), `Mix.Dep`, `Mix.SCM`/`Mix.SCM.Git`, `Mix.Shell`, `Mix.Release`, `Mix.Generator`, plus each `Mix.Tasks.*` module's `run/1`. `Mix.State`, `Mix.TasksServer`, `Mix.ProjectStack`, `Mix.PubSub`, `Mix.Sync.Lock` are internal (`@moduledoc false`) but are process-global singletons every task depends on.

## Trust / contract boundaries
The dominant boundary is **the project itself executes as code**, not parsed as data:
- `Mix.CLI.load_mix_exs/2` **compiles and runs `mix.exs`** (`Code.compile_file`). `Mix.Dep.Lock.read/1` **`Code.eval_quoted`s `mix.lock`**. Both mean cloning/building an untrusted project is arbitrary-code-execution by design — a per-file reviewer should not flag "we eval the lockfile" as a novel bug, but *should* watch for the guards around it (merge-conflict check, `%{}` shape match, `_ -> %{}` fallback swallowing real errors).
- **Operator/CLI-controlled:** `args`, env vars (`MIX_ENV`, `MIX_TARGET`, `MIX_DEBUG`, `MIX_PROFILE`, `MIX_OS_CONCURRENCY_LOCK`, `MIX_EXS`). Note `String.to_unsafe_atom`/`List.to_unsafe_atom` on env and task-name input in `Mix.CLI`/`Mix.State` — atom creation from external strings.
- **Network/peer-controlled:** dependency contents fetched by `Mix.SCM.Git` / Hex / rebar — attacker-authored `mix.exs`, source, and `rebar.config`.
- **Filesystem/other-process-controlled:** `Mix.Sync.Lock` and `Mix.Sync.PubSub` read files and bytes written by *other OS processes* (possibly crashed/stale, possibly an unrelated process that inherited an ephemeral port). This is the one place with genuine untrusted-binary decoding.

## Failure-mode model
Because most inputs arrive as already-parsed Elixir terms, the classic wire-decoder crashes are rarer; the characteristic Mix bugs are:
- **State-threading across the global singletons** — `Mix.ProjectStack` push/pop imbalance, `on_clean_slate`/`clear_stack` restoring the wrong prior state, `Mix.TasksServer` `insert_new` vs `insert` (re-run semantics) — a dropped/swapped state leaks between tasks silently.
- **Silent-swallow** — `else _ -> %{}` (lock read), `case {debug?, exception.mix}` in `Mix.CLI.run_task/2` only rescuing Mix-namespaced exceptions; wrong classification hides or reraises the wrong error and controls the process exit code.
- **Binary decode crashes** in `Mix.Sync.Lock` — `decode_lock_info/1`, `switch_file_read/1` use bare `<<...>> = data` matches with no error clause; a truncated/garbage lock file raises `MatchError` rather than being treated as stale.
- **Resource/cleanup leaks** — sockets and hard-link/`port_*`/`lock_N` files in `Mix.Sync.Lock`; the `try/after` in `with_lock` and `lock` is the safety net.
- **Deprecation/`cond` fallthrough** in `Mix.CLI` env/target resolution returning `nil` and thus the wrong environment.

## Common bug patterns to recognise
Atom-table growth from `to_unsafe_atom` on external strings; `function_exported?`-gated calls that assume arity; ordering-dependent side effects (the `take_over/3` comment stresses *precise order* — reordering the lock_0 replace / port_* removal / lock_N cleanup reintroduces the race); off-by-one in the switch-file segment math (`inactive_content_position`); `Enum.reverse` on an accumulator (subscriber `acc`) — a missing reverse or a missed nil-vs-list `acc` state delivers events out of order or drops them.

## Process / supervisor structure
`Mix.Supervisor` (`:one_for_one`, **`max_restarts: 0`**) supervises `Mix.Sync.PubSub`, `Mix.State`, `Mix.TasksServer`, `Mix.ProjectStack`. Because `max_restarts: 0`, **any crash of these singletons brings down the whole Mix instance** — they hold ETS/`persistent_term`/stack state that is not rebuildable mid-run. `Mix.PubSub` is a nested supervisor (`Subscriber` + a later-started `ListenerSupervisor`). `Mix.Sync.Lock` spawns a linked `accept_loop` process per lock. `Mix.ProjectStack`/`Mix.PubSub.Subscriber` are the gen_servers; `Mix.TasksServer` is an `Agent`.

## Cross-file invariants
- `Mix.State`/`TasksServer`/`ProjectStack` ETS tables exist for the whole run — every task assumes `Mix.start()` ran; readers use raw `:ets`/`:persistent_term`, bypassing the owning process, so they see writes without serialization (races are possible but tables outlive individual callers).
- `Mix.CLI` guarantees `loadconfig` runs before any task and that `mix.exs` is compiled before `Mix.Project.get()` is consulted — downstream tasks assume the project is already on the `ProjectStack`.
- `Mix.Sync.Lock` assumes it is the *only* writer of a given lock file at a time and that `Mix.Utils.detect_user_id!` partitions lock dirs per user; correctness of `switch_file_replace!`'s atomicity rests on that single-writer assumption, not enforced in-file.
- `Mix.PubSub.Subscriber` accumulates events (`acc` list) until `start_listeners` → `flush`; a listener started before flush, or a second flush, breaks the "deliver each event once, in order" contract.