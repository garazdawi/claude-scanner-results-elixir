# Cross-file findings

## Asserted-dependency ledger

Every `depends on` / out-of-scope contract asserted across the report set, resolved against the cited file's own report. (Rows the source itself discharged by *reading* the neighbour are marked Confirmed; rows where the cited file's report substantiates the contract are Confirmed; rows where the cited file's report says the opposite are Contradicted; rows nobody checked are Unresolved.)

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| app_loader.ex | compile.all/compile.ex/elixir.ex/dep-loader callers reach `read_app`/`read_cache` | Confirmed | Source read caller sources; callers' reports agree they invoke on every compile/deps |
| app_loader.ex | `:application.load` never runs `:mod`; `.app` path from operator `mix.exs` | Confirmed | Reasoned from OTP semantics; consistent with compile.app.ex report |
| cli.ex | dispatched tasks (deps.get/archive.install/compilers) own untrusted-data handling | Confirmed | Each dispatched task's report owns its sinks (archive.install, deps.get, etc.) |
| cli.ex | `String.to_unsafe_atom`; `Mix.Utils`/`Mix.Project` trusted | Confirmed | Fresh-VM-per-invocation; trusted-infra per threat model |
| compilers-elixir.ex | `_build` manifest = trust boundary; old-format branches; `@on_load` | Confirmed | `_build`-write == RCE invariant (THREAT_MODEL); manifest sinks own-file |
| compilers-erlang.ex | `Mix.Sync.PubSub.broadcast` short-circuit; `String.to_unsafe_atom`; hardened receive decoder | Confirmed | Source verified broadcast short-circuit + atom def; pubsub receive owns decode |
| compilers-protocol.ex | can a path-dep ship a consolidatable protocol atom with `..` to write sink | **Unresolved** | Report "could not settle"; no neighbour report resolves protocol-atom traversal |
| compilers-protocol.ex | `Macro.inspect_atom` escapes control chars; `_build` boundary | Confirmed | Escaping verified in-file |
| compilers-test.ex | `Mix.Compilers.Elixir.read_manifest` decode; `_build` boundary | Confirmed | Decode sink owned by elixir.ex report; trust-boundary per threat model |
| config.ex | stdlib `Config` RCE-by-design; no in-repo caller wires untrusted data | Confirmed | Full-tree caller grep done in-file |
| dep-converger.ex | `remote`/`scm` pre-resolved modules; loader/lock/Version own their sinks | Confirmed | Converger consumes structured output; sinks owned by named files |
| dep-elixir_scm.ex | callers (`loadpaths.ex:61`, `loader.ex:431`) only `!=`-compare decoded term | Confirmed | loadpaths.ex report confirms decode gates recompile only, not applied |
| dep-fetcher.ex | rebar `:file.consult` mints `..`-bearing app atom; loader guards only `is_atom` | Confirmed | Source-read rebar.ex + loader.ex; underlies loader's High traversal finding |
| dep-fetcher.ex | git arg-injection lives in `Mix.SCM.Git`; lock eval keys ≠ `app` | Confirmed | scm-git.ex & lock.ex reports own those sinks |
| dep-loader.ex | `Mix.SCM.Git.checkout/1` `File.rm_rf!(opts[:checkout])` completes traversal | **Contradicted** | scm-git.ex report *rules out* this same sink as "Mix-internal, safe" → Finding 1 |
| dep-loader.ex | Path/Hex SCMs consume same traversing `dest`; Hex validates names | Unresolved (partial) | Only Git exercised; Path/Hex `accepts_options` normalization not audited |
| dep-lock.ex | call sites (`mix.ex`, `elixir.ex:314`, `converger.ex:88`, `deps.*`) all call `read/1` | Confirmed | Lockfile-eval is a Finding; call sites confirmed in those reports |
| dep-umbrella.ex | `apps_paths` atoms operator-declared / `File.ls`-validated; atom sink in project.ex | Confirmed | Source-traced; project.ex owns the `to_unsafe_atom` site |
| dep.ex | `loader.ex:484 Path.wildcard` supplies dep-controlled `.app` name to `format_status` | Confirmed | Source-cited; underlies dep.ex's own escape-injection finding |
| dep.ex | `format_status` output flows to `shell.error/info` print sinks | Confirmed | Traced to fetcher.ex:176, deps.loadpaths.ex:201, deps.ex:233 |
| exceptions.ex | `Mix.NoTaskError` sites feed developer task names; `command_to_module_name` returns string | Confirmed | Every raiser traced; utils.ex:675 returns string (no atom) |
| generator.ex | only two internal `copy_template` callers pass operator `rel/` paths | Confirmed | grep + release.ex read |
| hex.ex | fetch/`--sha512` gating in local.hex/local.rebar; install integrity their concern | Confirmed | Consumers' reports own those sinks |
| local-installer.ex | `Mix.Utils.read_path`→checksum before payload; httpc_response no raw peer bytes | Confirmed | Source-read utils.ex:803-848 / 940-944 |
| local-installer.ex | atom sinks fed operator argv; git/zip/read_httpc sinks in other files | Confirmed | archive.install & utils reports confirm |
| local.ex | `read_httpc` TLS verify_peer; body-size cap belongs there | Unresolved (partial) | Size-cap not confirmed by utils.ex report |
| local.hex.ex | `find_by_elixir_version`/`checksum` fail-closed shapes | Confirmed | Source-read local.ex:219-228, utils.ex:832-848 |
| local.hex.ex | `read_httpc` validates/encodes CSV path bytes into request target | **Unresolved** | utils.ex report never addresses CSV-byte→request-target validation |
| local.rebar.ex | `read_httpc` retries keep verify_peer; checksum before write; fixed target path | Confirmed | Source-read utils.ex in full |
| new.ex | Generator/Project/Macro/`to_unsafe_atom` guard behaviours | Confirmed | Verified behaviourally against toolchain |
| profile.{cprof,eprof,fprof,tprof}.ex | `Mix.Tasks.Run.run/5` sinks driven by operator argv; `parse_mfa` parse-not-eval | Confirmed | run.ex & utils.parse_mfa read/executed |
| profile.*.ex | OTP `:cprof`/`:fprof`/`:tprof` trusted infra | Confirmed | Trusted-infra per threat model |
| release.ex | `Mix.Release.copy_app`/`from_config!` consume hostile dep `vsn` into path | Confirmed | release.ex report owns this (validation-gap finding); sink in same-folder tasks/release.ex |
| release.ex | `mix compile` precedes assembly → dep already had RCE | Confirmed | In-file ordering; narrow never-compiled-dep exception noted |
| release.init.ex | `Mix.Tasks.Release` renders templates; `release.name` from operator config | Confirmed | Source-read release.ex |
| remote_converger.ex | `Mix.State` fixed-key insert; `converger.ex` only reads stored module | Confirmed | Source-read state.ex + converger.ex |
| run.ex | `app.start`/`loadconfig` own dep-code exec / config load | Confirmed | Those reports own the sinks; run.ex forwards fixed name |
| scm-git.ex | `Mix.Dep.Lock.read/1` returns eval'd rev map unvalidated; `put_lock` attaches raw | Confirmed | Source-read dep/lock.ex:25-36 + converger.ex:253-255 |
| scm-git.ex | `System.cmd` single-argv; git permutes options after positionals | Confirmed | Executed (arg-injection basis) |
| scm-path.ex | dangerous `:dest` consumption in loader/deps.compile/converger; transitive-path honoring | Unresolved (partial) | Explicitly not audited; transitive-path-dep confinement question open |
| scm.ex | `Mix.State.update/2` non-atomic public-ETS RMW; `:scm` init list | Confirmed | Source-read state.ex; loader.ex:224 sole consumer |
| scm.ex | fetch/arg-injection/checksum in Git/Utils/archive/converger | Confirmed | Named files own those sinks |
| shell-io.ex | `Mix.Shell.cmd/3` binary→`System.shell` sink; `printable_app_name` raw atom | Confirmed | Source-read shell.ex + project_stack.ex |
| shell-io.ex | caller sites cross the trust boundary (not enumerated) | Unresolved | Callers not enumerated; deferred |
| shell-process.ex | `cmd/3` holds the sink; `$callers` set by trusted runtime | Confirmed | Source-read shell.ex; runtime-trust scope |
| shell-quiet.ex | `cmd/3` dispatch; delegates error/prompt to io.ex | Confirmed | Read + executed dispatch; io.ex read |
| shell-quiet.ex | callers decide whether attacker bytes reach `cmd(binary)` | Unresolved | Scope pointer to Cmd/Deps.Compile/SCM callers |
| shell.ex | io.ex & quiet.ex forward `command` to cmd/3 without interpolation | Confirmed | Both read in full |
| shell.ex | deps.compile.ex chooses to forward dep-controlled strings into shell branch | Confirmed | deps.compile.ex report confirms `compile:` = by-design |
| state.ex | callers may forward attacker bytes as ETS keys in a loop | Unresolved | Callers not audited ("out of review scope") |
| state.ex | dep load paths added after builtin_apps snapshot | Confirmed | Executed reproduction |
| sync-lock.ex | `Mix.Utils.detect_user_id!` integer uid, no secret component | Confirmed | Source-read utils.ex:1023-1048 |
| sync-lock.ex | `File.open!`/`rm` symlink/OS semantics | Confirmed | Executed end-to-end |
| sync-pubsub.ex | `Mix.Utils.non_executable_binary_to_term` no `:safe` → interns atoms | Confirmed | Executed; is the utils.ex finding's other half |
| sync-pubsub.ex | callers `Deps.Compile`/`Task.Compiler` invoke `broadcast` on compile path | Confirmed | task.compiler.ex report confirms publish-on-compile |
| task.compiler.ex | `command_to_module`→`Module.concat` fed config names; pubsub publish-only | Confirmed | Traced; pubsub.ex + utils.ex read |
| task.ex | `command_to_module/2`→`Module.concat` mints permanent atom even if unloaded | Confirmed | Read + executed (delta=3001) |
| task.ex | `Mix.TasksServer`/`Mix.State` public ETS co-tenant-poisonable | Confirmed | Threat-model out-of-scope attacker |
| tasks_server.ex | `Mix.Task` `to_unsafe_atom(task)` mints atoms from alias keys | Confirmed | task.ex report owns this atom sink (Low) |
| app.config.ex | `loadconfig.load_runtime` reads only its `file` arg, imports disabled | Confirmed | Source-read loadconfig.ex:67-71 |
| app.config.ex | `Config.Reader.read!` executes operator file | Confirmed | Config-as-code by design (input-side sufficed) |
| app.start.ex | `Mix.Shell.IO.error` emits verbatim; `Mix.Dep.Umbrella` app atoms operator-owned | Confirmed | Read + executed shell/io.ex; umbrella read |
| app.tree.ex | `write_dot_graph!`/`--output` operator CLI; `ensure_loaded` no start/2 | Confirmed | Read to needed extent; executed |
| archive.build.ex | `Mix.Local.archive_name` rootname confinement; symlink-following load-bearing | Confirmed | Read local.ex + project.ex + utils.symlink_or_copy |
| archive.build.ex | extraction-side traversal defense in archive.install | Confirmed | archive.install.ex owns it (and finds a bypass) |
| archive.check.ex | `Application.load` reads only spec; `Version.parse_requirement` safe | Confirmed | Executed |
| archive.check.ex | archive planting / `.ez` extraction safety in archive.install/Local/Utils | Confirmed | Those reports own it |
| archive.ex | hostile filename lands only via archive.install (auto-loads = RCE) | Confirmed | RCE-domination; archive.install owns sanitization |
| archive.install.ex | OTP `zip.erl` LH/CD name differential, net-level check, no make_symlink | Confirmed | Read zip.erl + executed end-to-end (is the High finding) |
| archive.install.ex | `read_path` verifies `--sha512` before install/3; installer atom sinks = operator argv | Confirmed | Read installer.ex:238-287 |
| archive.uninstall.ex | `Installer.uninstall/4` operator-argv `name`→`Path.join`→`rm_rf!` | Confirmed | Read in full; grep sweep of callers |
| clean.ex | `command_to_module`→concat fed operator config; `:compilers`/build_path operator config | Confirmed | Executed atom-delta; read definitions |
| cmd.ex | `Mix.Shell.cmd/3` shell vs argv; no internal caller forwards untrusted data | Confirmed | Executed both; grep |
| compile.all.ex | `Mix.AppLoader` .app parse/cache serialization sink | Confirmed | app_loader.ex report owns it (with `_build` boundary) |
| compile.app.ex | `to_unsafe_atom` per .beam name; `:file.consult`/`:application.load` no eval | Confirmed | Verified vs /opt/elixir source + executed |
| compile.elixir.ex | `Mix.Compilers.Elixir` performs compile/manifest/beam safely | Confirmed | Out-of-scope but elixir.ex report owns those sinks |
| compile.erlang.ex | `Mix.Compilers.Erlang` confines paths, gates manifest decode to `_build` | Confirmed | Read erlang.ex in full |
| compile.leex/yecc.ex | `Mix.Compilers.Erlang` path confinement + manifest decode gating | Confirmed | erlang.ex read to confirm flow |
| compile.protocols.ex | `Mix.Task.__using__` injects no runtime sink; consolidation in Compilers.Elixir | Confirmed | Read task.ex:139-153 |
| deps.clean.ex | `Mix.Dep.Converger.converge/1` returns `%Mix.Dep{}`; struct reaches no atom/rm sink | Confirmed | Read dep.ex:57-81; traced in-file uses |
| deps.compile.ex | loader builds struct guarding only `is_atom(app)`, no `..` rejection | Confirmed | Confirms loader's no-validation (backs Finding 1) |
| deps.compile.ex | `File.rm_rf!` app-name traversal "adds no capability beyond by-design RCE" | **Contradicted** | loader.ex proves traversal fires pre-code-exec, attacker-controlled → Finding 1 |
| deps.compile.ex | `Mix.Shell.cmd/3` shell vs `System.cmd` argv | Confirmed | Read shell.ex:164 |
| deps.get.ex | `Fetcher.all`/`Lock.read`/`archive.check` handle fetch/checksum/zip/lock safely | Confirmed | Those reports own the sinks |
| deps.get.ex | internal callers pass no hostile `--only`/`--target` | Confirmed | Read both call sites |
| deps.loadpaths.ex | `Mix.Dep.format_status` clause coverage; `:divergedreq` raw vsn/app | Confirmed | Read dep.ex clauses (raw-interp is dep.ex's finding) |
| deps.loadpaths.ex | scm `fetchable?/0` predicates; `load_paths/1` path construction | Confirmed (partial) | scm files read; `load_paths` traversal deferred to dep.ex |
| deps.partition.ex | `ProjectStack.push`/`CLI.main`/`compile_single` are build-controlling | Confirmed | Standard Mix behaviour (backs its own RCE finding) |
| deps.precompile.ex | real work in project-aliased tasks | Confirmed | Full read: run/1 is a no-op |
| deps.tree.ex | `Mix.Utils` DOT/terminal escaping + `--output` write safe | **Contradicted** | utils.ex tree-printer does NOT strip ANSI/control → Finding 3 |
| deps.tree.ex | Converger/Loader perform mix.exs RCE + atom creation | Confirmed | Those reports own it |
| deps.unlock.ex | `Lock.read`/`write`, `Converger.converge`; lock keys already atoms | Confirmed | Lock-eval owned by lock.ex; in-file uses benign |
| deps.update.ex | Lock/Fetcher/RemoteConverger/SCM.Git own atomization/network/git-ref | Confirmed | Those reports own the sinks; grep found no internal callers |
| deps.ex | `Mix.Dep.format_dep`/`format_status`/`format_lock` neutralize terminal control chars | **Contradicted** | dep.ex proves `format_status` interpolates raw (escape-injection) → Finding 2 |
| deps.ex | `OptionParser` non-strict atom creation (CLI boundary) | Confirmed | Verified behaviour |
| do.ex | `Mix.Task.run`/`run_in_apps`/`OptionParser.parse_head!` own dispatch | Confirmed | grep: callers are CLI dispatcher + tests |
| escript.build.ex | `Mix.Task.run("compile")` runs before dep-influenced code | Confirmed | In-file ordering read |
| escript.build.ex | `Mix.Release.strip_beam`/`:escript.extract` safe on malformed .beam | Unresolved | Explicitly not assessed |
| escript.build.ex | `Path.wildcard` `**` symlink traversal (priv exfil) | **Unresolved** | Asserted, not empirically confirmed; no neighbour resolves it |
| escript.ex | `escript.install` writes hostile filenames; `name_for`/`URI.parse` basename | Confirmed | Executed (backs its own finding) |
| escript.install.ex | `Installer` sets basename before install/3; `read_path` verifies URL `--sha512` | Confirmed | Read installer.ex:128 + utils.ex:809/822 |
| escript.install.ex | `read_httpc` TLS; installer atom sinks; SCM.Git — inherited sinks | Confirmed (partial) | read_httpc reviewed "only enough to clear URL flow" |
| escript.uninstall.ex | `Installer.uninstall/4` operator-argv path ops | Confirmed | Read uninstall/4 in full |
| eval.ex | `app.config` evaluates operator's own mix.exs/config (by-design) | Confirmed | In-file forwarding traced |
| format.ex | Myers O(N·D); manifest `_build`-local; plugin/config run via compile | Confirmed | Executed worst-case + threat-model |
| help.ex | `IO.ANSI.Docs`/`Mix.Shell`/`Code.fetch_docs` render doc content WITHOUT stripping control seqs | Confirmed | End-to-end flow verified; is a systemic-escape data point (see Finding 3 note) |
| help.ex | `IEx.Introspection.decompose` parses but does not evaluate | Unresolved | Judged non-evaluating; external file not audited |
| iex.ex | `Mix.raise/1,2` forwards verbatim, no format interpretation | Confirmed | Read mix.ex:623-648 |
| loadconfig.ex | `App.Config`/`preserve_config`/`--config`/installer callers use operator paths | Confirmed | Read/grep-confirmed; Mix never loads a dep's config |
| loadconfig.ex | `Config.Reader.read!` executes operator file only | Confirmed | Input-side sufficed (by-design) |
| loadpaths.ex | `Mix.Project` path builders from operator config; `ElixirSCM` decode gated to `_build` | Confirmed | Read project.ex + elixir_scm.ex |
| loadpaths.ex | `deps.loadpaths` task owns forwarded `args` | Confirmed | deps.loadpaths.ex report owns it |
| project.ex | `Mix.CLI` state_loader = operator argv; `ProjectStack` merge/drop defense | Confirmed | Read cli.ex + project_stack.ex (merge order verified) |
| project.ex | archive/git/dep/utils sinks live in those files | Confirmed | Scoping deferral; those reports own them |
| project_stack.ex | project's code evaluated *before* config/file pushed to `push/3` | Confirmed | project_stack.ex ruled push/3 out on this basis; project.ex substantiates push mechanics |
| pubsub-subscriber.ex | `Sync.PubSub` loopback TCP discoverable → flood unbounded buffer | Confirmed | Reasoned from reading sync/pubsub.ex (labeled asserted; consistent with utils finding) |
| pubsub-subscriber.ex | `start_listeners` `whereis` guard prevents double-flush; listeners trusted | Confirmed (partial) | Crash verified; guard behaviour asserted |
| pubsub.ex | `Supervisor.start_child` serialises; `:listeners` root-config opt-in | Confirmed | Executed OTP-29 contract; cross-file read |
| task-compiler.ex | pubsub deserialization guarded by `non_executable_binary_to_term` | Confirmed | Read pubsub.ex + utils.ex:30-34 |
| test.coverage.ex | OTP `:cover.import` unsafe `binary_to_term` (the caller finding) | Confirmed | OTP does no validation; deser executed |
| test.ex | `Compilers.Test`/`ExUnit.Filters`/`Test.Coverage` own require/atom/export sinks | Confirmed | Those files/apps own the sinks |
| will_recompile.ex | `reset_config_mtime` constant; `apps_paths` validated atoms; atom sink in project.ex | Confirmed | Read project_stack.ex + project.ex |
| xref.ex | `parse_mfa` parse-not-eval; `read_manifest` `_build` first-party; `--output` operator argv | Confirmed | Read utils.ex + elixir.ex + write helpers |
| mix.ex | Deps/SCM/Utils/Lock own archive/git/checksum/TLS/lock-eval; `Mix.install` doesn't weaken | Confirmed | Those reports own the sinks |
| mix.exs | `Mix.Project.__using__` wires after-compile only; no sink reached | Confirmed | Read project.ex:239-249; traced outbound calls |

## Findings

#### `CSCAN-crossfile-dep-app-name-path-traversal-underdismissed`

- **Finding ID** — `CSCAN-crossfile-dep-app-name-path-traversal-underdismissed`
- **Severity** — High
- **Confidence** — high
- **Files involved** — `lib-mix-dep-loader.ex.md`, `lib-mix-scm-git.ex.md`, `lib-mix-tasks-deps.compile.ex.md`, `lib-mix-dep-fetcher.ex.md`
- **Issue** — Two files dismiss the exact filesystem sink that `dep/loader.ex` proves is exploitable. `scm/git.ex`'s report **rules out** `File.rm_rf!(opts[:checkout])` (git.ex:107-108) on the grounds that "the checkout path is computed by Mix (`deps/<app>`), not supplied through this file's API." `deps.compile.ex`'s report **rules out** the same-shape `File.rm_rf!(Path.join([build_path, "lib", Atom.to_string(dep.app)]))` (deps.compile.ex:129), arguing an `..`-bearing app name "adds no capability beyond the by-design RCE" because "that dependency already has arbitrary code execution via `rebar.config.script`." Both dismissals rest on an invariant that `dep/loader.ex`'s verified High finding (`CSCAN-loader-dep-app-name-traversal`, loader.ex:182) contradicts: a transitive rebar dependency's `rebar.config` supplies its sub-dependency **name** as pure data (`:file.consult`, no code run), that name is joined unconfined into `dest`/`:checkout` with no `..`/segment check (loader guards only `is_atom(app)`), and the delete+write detonates in `Mix.SCM.Git.checkout/1` **at `mix deps.get` time, before any dependency code executes** (reproduced end-to-end: `'../../PWNED_SENTINEL'` deleted files outside the project tree). So `opts[:checkout]` is attacker-controlled, and the "dep already had RCE" argument is false for the pre-fetch delete. The net vulnerability is not missed — `loader.ex` owns it — but the two neighbours' "ruled out" verdicts are locally unsound, and a reader trusting either in isolation would conclude the sink is safe.
- **Resolution** — Treat `loader.ex`'s finding as authoritative and correct the two dismissals: the app-name/`:checkout`/`build` path component must be validated (`Path.safe_relative` / single-segment check) at the point it is built in `with_scm_and_app/5`, not trusted downstream. No further source reading is required — the contradiction is already established across the three reports; the fix belongs in `dep/loader.ex` per its own suggestion.

#### `CSCAN-crossfile-format-status-terminal-escape-deferred`

- **Finding ID** — `CSCAN-crossfile-format-status-terminal-escape-deferred`
- **Severity** — Low
- **Confidence** — high
- **Files involved** — `lib-mix-tasks-deps.ex.md`, `lib-mix-dep.ex.md`
- **Issue** — `deps.ex`'s report dismisses terminal/log injection at its `shell.info`/`shell.error` sinks (deps.ex:227-238) by explicitly **deferring the escaping obligation** to the string constructors: "any escaping obligation for dependency-derived content (e.g. a transitive dep's `:branch`/`:tag` name) belongs to those constructor functions [`format_dep/1`, `format_status/1` in `Mix.Dep`], not to this printer." `dep.ex`'s report shows that obligation is **not met**: `CSCAN-dep-noappfile-escape-injection` (dep.ex:304, verified) proves `format_status/1`'s `{:noappfile, {path, other_path}}` clause interpolates a dependency-controlled `.app` basename **raw** (`try changing the dependency name to :#{other_app}`) while the adjacent line escapes the sibling value via `inspect`, so ANSI/control bytes survive into exactly the `shell.error`/`shell.info` line `deps.ex` prints. The two reports directly contradict on whether `format_status` neutralizes dependency-derived control bytes.
- **Resolution** — Already resolved by reading both reports: the escape sink is real and owned by `dep.ex`. The fix (consistent `inspect`/control-byte stripping in `format_status/1`) is in `dep.ex`'s suggested-fix; `deps.ex`'s deferral should be re-labelled from "ruled out" to "inherits a confirmed Low finding in `Mix.Dep`."

#### `CSCAN-crossfile-tree-printer-dep-name-escape-reachability`

- **Finding ID** — `CSCAN-crossfile-tree-printer-dep-name-escape-reachability`
- **Severity** — Low
- **Confidence** — medium
- **Files involved** — `lib-mix-utils.ex.md`, `lib-mix-tasks-deps.tree.ex.md`, `lib-mix-tasks-xref.ex.md`
- **Issue** — `utils.ex`'s report raises `CSCAN-utils-tree-printer-terminal-escape` (utils.ex:518) as a **low-confidence, `asserted`** finding precisely because it could not trace a caller forwarding attacker-controlled data into `print_each_node/5`: "the tree callbacks are supplied by task modules not in this file … I could not trace a concrete attacker-controlled string." `deps.tree.ex`'s report supplies the missing half: it confirms a malicious dependency's `requirement`/app metadata **is** dependency-controlled and reaches `print_tree`/`Mix.Utils.print_each_node` unescaped (deps.tree.ex ruled-out entry: "a malicious dependency's `requirement`/app metadata is dependency-controlled … the only sinks are in `Mix.Utils`/`Mix.Shell`"), dismissing it only because dependency-`mix.exs`-RCE dominates. So the utils hedge is resolved to *reachable*: the two reports together confirm the escape-injection path exists (utils has no ANSI stripping; deps.tree feeds it dependency-controlled bytes). Both agree severity is Low because a hostile dependency already achieves RCE when its `mix.exs` is evaluated. `xref.ex` is *not* a live caller of the same sink (its report verifies the printed values are first-party file/module names).
- **Resolution** — Upgrade `utils.ex`'s finding from `asserted`/low-confidence to confirmed-reachable (via the `deps.tree` caller), keeping Low severity. This is one instance of a folder-wide pattern (below); the durable fix is to strip control bytes in `Mix.Utils`'s tree/JSON/DOT writers rather than at each caller.

### Folder-level pattern (already-owned, noted once)

**Unsanitized dependency-controlled strings reach the terminal via `Mix.Shell`.** The same sink class recurs across `dep.ex` (`format_status` raw interpolation — owned finding), `utils.ex` (tree/JSON/DOT printers — owned finding), `deps.tree.ex` (forwards dependency `requirement`/name), `escript.ex` (hostile escript filename printed), and `help.ex` (`IO.ANSI.Docs` renders third-party `@moduledoc`/`@shortdoc` without stripping control sequences). Individually each is Low (dependency/archive content that reaches these printers has typically already run as code), but the systemic root cause is that Mix has no central control-byte sanitizer on its shell-output path; `Mix.Shell.IO` emits `IO.ANSI.format` output verbatim (confirmed in `shell-io.ex` / `app.start.ex` reports). A single sanitizer at the `Mix.Shell` boundary would close all of them; the per-file findings in `dep.ex` and `utils.ex` already track two of the instances.
