# Cross-file findings

The per-file reports in this set do **not** use the machine-extractable `* Depends on: <file/symbol> — <contract>` format. `grep -nrh "^[[:space:]]*\* Depends on:"` returns zero rows across the entire set. Every cross-file dependency is instead expressed as prose ("depends on", "out of scope", "trusted", "assumed", "relies on"). The ledger below enumerates each such asserted cross-file dependency and resolves it against the cited neighbour's own report.

## Asserted-dependency ledger

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| app_loader.ex | `Mix.Project`/`Mix.Dep`/`Mix.State` global state for `load_apps`/`traverse_apps` control flow | Unresolved | Neighbour state not stood up; crash mechanism executed but reachability only reasoned. |
| app_loader.ex | future/historical `@manifest_vsn` change yielding a mismatched-but-valid term | Confirmed | Self-contained; crash verified, trigger is an upgrade-time event. |
| dep/converger.ex | file is `IDENTICAL` to installed converger; `converger_test.exs` passes | Confirmed | Self-substantiated by diff + passing test suite. |
| dep/converger.ex | `all/9` recursion / `Mix.Dep.Loader` cross-file invariant not broken | Confirmed | dep/loader.ex report finds no defect and is a faithful upstream copy. |
| local.ex | upstream signature verification of the S3 CSV before `find_matching_versions!` | Unresolved | No caller report (local.rebar/local.hex) describes CSV signature verification; inherited-risk gap. |
| project.ex | dep-convergence invariant for `traverse_deps_depth/5` termination | Confirmed | Upgraded to executed (diamond + self-cycle) within its own report. |
| pubsub.ex | `Mix.PubSub.Subscriber` accumulator/flush ordering ("deliver once, in order") | Confirmed | pubsub/subscriber.ex Ruled-out: prepend-then-reverse ordering executed correct, single flush. |
| tasks/app.config.ex | `config[:config_path]` always populated (`Path.dirname` never gets `nil`) | Unresolved | No cited report verifies `config_path` is non-nil; taken on faith. |
| tasks/archive.ex | `Mix.path_for/1` returns binary; `Mix.Shell.info/1` accepts ansidata | Confirmed | Self read of mix.ex + shell.ex within the report. |
| tasks/clean.ex | `Mix.Project`/`Mix.Task.Compiler`/`Mix.Compilers.Protocol` runtime state | Unresolved | Reasoned from helper semantics, not driven end-to-end. |
| tasks/compile.erlang.ex | `Mix.Compilers.Erlang` `parallel`/`compile_entries` interaction | Confirmed | Self read of compilers/erlang.ex:107-190; interaction verified consistent. |
| tasks/compile.leex.ex | erlang.ex callback 3-tuple contract; yecc sibling structural identity | Confirmed | Self read of erlang.ex + compile.yecc.ex. |
| tasks/compile.yecc.ex | erlang.ex `compile/6` `(input,output)` + 3-tuple return | Confirmed | Self read of erlang.ex:76-98, 282-308. |
| tasks/deps.compile.ex | `Deps.Partition.server/3` returns truthy iff recompile needed | Confirmed | Self read partition.ex:10-119; partition.ex report does not contradict return-usage. |
| tasks/loadconfig.ex | `Mix.Task.run/2 → run/1` passes token list as `args` | Unresolved | Not driven end-to-end; CaseClause reproduced only in isolation. |
| tasks/loadpaths.ex | `Mix.Dep.ElixirSCM.read/0` tuple contract | Confirmed | dep/elixir_scm.ex enumerates `read/1` return incl. fallback tuple. |
| tasks/loadpaths.ex | `Mix.Project.app_path/consolidation_path/compile_path` shapes | Unresolved | Explicitly "took as trusted"; no report verifies these helpers. |
| tasks/local.rebar.ex | `Mix.Local.find_by_elixir_version/3` tuple order (local.ex:226) | Confirmed | Positional match verified against local.ex literal return. |
| tasks/release.ex | `Mix.Tasks.Release.Init` EEx templates reference only in-scope assigns | Confirmed | Self read release.init.ex:46-513; release.init.ex report agrees. |
| tasks/release.ex | `Mix.Release.*` delegate internals | Unresolved | Declared a genuine other-file boundary; delegate internals not cross-checked. |
| tasks/will_recompile.ex | `Mix.ProjectStack.reset_config_mtime/0` return shape | Confirmed | Self read project_stack.ex:126-134. |
| mix.ex | `Mix.install/2` orchestration over ProjectStack/Task/Dep.Lock | Unresolved | Not executed (needs network/dep fetch). |
| tasks/app.start.ex | `Application.ensure_all_started/2` `{:error,{App,Reason}}` shape | Unresolved | Asserted from OTP docs; not run in `:serial`/`:concurrent`. |
| shell.ex | `Mix.ProjectStack.printable_app_name/0` internal correctness | Unresolved | Explicitly not audited; project_stack report does not name it. |
| scm.ex | `Mix.State.update/2` non-atomic read-modify-write | Confirmed | Self read state.ex:41-44; minted as finding in scm.ex (state.ex report silent, not contradictory). |
| tasks/deps.tree.ex | `Mix.Dep.Converger.converge/1` one-dep-per-app dedup invariant | Unresolved | Assumed; converger property not driven for `find_dep`. |
| tasks/deps.update.ex | `Mix.Dep.Fetcher.by_name/4` old/new lock arg order (fetcher.ex:28) | Confirmed | Positional match verified. |
| tasks/compile.all.ex | documented contracts of ~6 compiler collaborators | Unresolved | Explicitly not re-audited. |
| tasks/compile.elixir.ex | `Mix.Compilers.Elixir`/`OptionParser` | Unresolved | Out of scope, not audited. |
| dep/fetcher.ex | `Converger.converge/4` returns `{all_deps,apps,new_lock}`; cycles pre-rejected | Confirmed | converger.ex report finds no defect; faithful upstream copy. |
| task.ex | `Mix.TasksServer` + `Mix.ProjectStack` state-threading | Confirmed | Self read both; tasks_server.ex confirms insert_new/member semantics. |
| hex.ex | `Local.Hex.run → Archive.Install → Installer.install` boolean chain | Confirmed | Self read of full delegate chain (all branches return boolean). |
| tasks/profile.tprof.ex | `Mix.Tasks.Run.run/5` forwards script tail to `System.argv` | Confirmed | Self read run.ex:120-174. |
| tasks/source.ex | `IEx.Introspection` clause coverage | Unresolved | Outside reviewed folder; not audited. |
| tasks/deps.partition.ex | local process on the ephemeral loopback port is out-of-scope ("not exercised by valid inputs") | **Contradicted** | THREAT_MODEL designates cross-OS-process sockets untrusted, and sync/pubsub.ex mints exactly this adversary class — see finding below. |

## Findings

#### `CSCAN-crossfile-partition-tcp-untrusted-peer-unscoped`

- **Finding ID** — `CSCAN-crossfile-partition-tcp-untrusted-peer-unscoped`
- **Severity** — Medium
- **Confidence** — medium
- **Files involved** — `lib-mix-tasks-deps.partition.ex.md`, `lib-mix-sync-pubsub.ex.md`, `THREAT_MODEL.md`
- **Issue** — Two files expose the *same* trust boundary — a Mix-opened loopback TCP listen socket that any local OS process can connect to — but their reports apply opposite scope rulings. `deps.partition.ex` opens `:gen_tcp.listen(0, ...)` and reads `:`-delimited lines from connected peers, feeding each segment through `String.to_unsafe_atom` and destructuring the result as a fixed two-element list (`[app, compiled?] = ...`, partition.ex:155-156). Its report **rules these out as out-of-scope**, writing that a non-conforming peer causing a `MatchError` on the destructure, or unbounded atom-table growth from crafted segments, "require a peer other than the trusted worker … a pre-existing localhost-race design property, not exercised by valid inputs. Untried — these axes stay asserted." Meanwhile `sync/pubsub.ex` — whose loopback socket is architecturally identical — treats precisely this local-process adversary as **in-scope** and mints a Medium finding (`CSCAN-pubsub-body-recv-infinite-timeout`), justifying it by the folder threat model: "the folder threat model explicitly designates `Mix.Sync.*` as the untrusted-binary surface reading bytes from other OS processes." The `THREAT_MODEL.md` invariant states that `Mix.Sync.Lock`/`Mix.Sync.PubSub` are "**the one place** with genuine untrusted-binary decoding" — but `deps.partition.ex`'s ephemeral control channel is a second such place the invariant overlooked, so `deps.partition.ex`'s dismissal rests on an incomplete invariant. Under the same standard `sync/pubsub.ex` (correctly) applied, the partition server's peer-controlled `MatchError` (server-loop crash) and `String.to_unsafe_atom` growth are reachable by a local process that races the trusted workers to the listen port, not merely a theoretical axis.
- **Resolution** — Read the actual source of `elixir/lib/mix/lib/mix/tasks/deps.partition.ex` around the `accept`/`server_loop` and message-parse path (lines ~101, 155-156) to confirm (a) that the listen socket is bound to loopback with no peer authentication and (b) whether a racing local connection can be accepted before the forked workers connect. If so, upgrade the two "asserted/untried" axes in `deps.partition.ex`'s Ruled-out section to a finding (guard the two-element destructure and avoid `String.to_unsafe_atom` on peer bytes), and reconcile `THREAT_MODEL.md`'s "one place" invariant to include the `deps.partition` control channel so future single-file reviews score both sockets by the same rule.
