# Mix bugfixes scan — ranked summary

**Scope:** `elixir/lib/mix`. No `VALIDATION.md` present, so severity/confidence come from each report's own (mostly execution-`verified`) labels. Cross-file (`CROSS_FILE.md`) and cluster findings are called out as such and ranked by impact alongside the per-file findings. `CROSS_FILE.md` notes the reports use prose dependencies rather than machine-extractable `Depends on:` lines; its ledger has one **Contradicted** and several **Unresolved** entries (listed at the end).

---

## 1. Compiler returning bare `:error` is silently downgraded to `{:noop, []}` — build not failed *(single-file, High)*
`lib-mix-task.compiler.ex` · `normalize/2` (task.compiler.ex:317-335). The `@callback` and moduledoc both sanction a compiler `run/1` returning the bare atom `:error`, but no clause matches it — it falls to the catch-all, which prints a spurious "was supposed to return …" warning and returns `{:noop, []}`. Consequences: (1) a compile **failure is reported as success/noop so the build is not failed**; (2) `:noop` instead of `:error` means following compilers keep running instead of halting as documented; (3) a correct compiler author is told they did it wrong. Bare `:ok`/`:noop` are handled — only `:error` was omitted. Verified in isolation; downstream build-not-failed effect reasoned from control flow (hence confidence medium). Highest-impact finding in the set.

## 2. `--cover` below-threshold coverage code (3) overrides a real test-failure exit status *(cluster, Medium)*
`CLUSTER_lib-mix-tasks-6` · test.ex ↔ test.coverage.ex. Both register competing `System.at_exit` hooks; because `cli.ex` folds hooks LIFO, the coverage hook's `exit({:shutdown, 3})` wins over test.ex's deliberately-chosen failure `exit_status` (default 2, or user `--exit-status N`). A failing suite run with `--cover` under threshold therefore **exits 3, not the test-failure code** — CI branching on the exit status is misled about whether tests actually passed. No single-file reviewer sees both hooks plus the fold order. (Same cluster also carries a Low doc/impl mismatch: test.ex docs say threshold exits status 1, code exits 3.)

## 3. `mix cmd` propagates sub-command failure with a bare-integer `exit`, losing the real exit code *(single-file, Medium)*
`lib-mix-tasks-cmd.ex:135`. On non-zero sub-command exit, `exit(status)` (bare int) is an *abnormal* exit: Elixir prints a crash dump/stacktrace pointing at cmd.ex and halts the OS process with code **1**, discarding the command's actual status. Every other Mix abort site uses `exit({:shutdown, code})`. Costs: confusing stacktrace on every failing `mix cmd` (linters, make, per-app umbrella commands), and CI/scripts always see `1` instead of the true status. Verified end-to-end. Fix: `exit({:shutdown, status})`.

## 4. `mix local.hex VERSION --if-missing` has the version comparison reversed *(single-file, Medium)*
`lib-mix-tasks-local.hex.ex:54`. Uses `Version.compare(Hex.version(), version) == :gt`. When the installed Hex is **older** than requested, this is false so the outdated Hex is **not upgraded** (the opposite of what `--if-missing` bootstrap automation expects); when installed is **newer**, it reinstalls/**downgrades**. Correct test is `== :lt`. Verified; confidence medium.

## 5. Cache version-tag mismatch crashes the build after a `@manifest_vsn` bump *(single-file, Medium)*
`lib-mix-app_loader.ex:13-21` · `read_cache/1`. The `else` clause of the `try` runs *outside* the `try` body, so `rescue _ -> []` does not catch it. A valid on-disk term with an old version tag — **exactly what every existing project has right after Mix bumps `@manifest_vsn` on upgrade** — raises `TryClauseError` instead of the intended silent rebuild (`[]`). Users must manually `mix clean`. Defeats the very mechanism the version tag exists for. Verified.

## 6. `mix deps.compile` crashes on empty/non-numeric `MIX_OS_DEPS_COMPILE_PARTITION_COUNT` *(single-file, Medium)*
`lib-mix-tasks-deps.compile.ex:103`. Unguarded `System.get_env(..., "0") |> String.to_integer()`. `System.get_env` returns `""` (not the default) when the var is *set-but-empty* — a common CI footgun (`export VAR=` to "disable" it) — so `String.to_integer("")` raises an opaque `ArgumentError` naming neither the variable nor the task. Reached on every `mix deps.compile` (incl. transitively via `mix compile`/`deps.get`/`Mix.install`). Verified.

## 7. Node-wide pub/sub hang: body `recv` uses an infinite timeout *(single-file, Medium)*
`lib-mix-sync-pubsub.ex:248` · `receive_message/1`. Header `recv` has a 1000 ms timeout, but the body `recv(socket, size)` defaults to `:infinity`. Any local process (or a publisher that crashes mid-send) can send a valid header claiming a large `size`, then stall — the single per-node receive loop blocks forever, so **every subsequent pub/sub message to that node is silently never delivered**, with no crash. Verified. (This file also has a Low monitor-leak-per-subscribe finding.) Related to the CROSS_FILE contradiction, item 18.

## 8. `deps.partition` worker death while idle raises `MatchError` and skips socket cleanup *(single-file, Medium)*
`lib-mix-tasks-deps.partition.ex:193` · `tcp_failed!/4`. When a worker socket closes/errors while the worker is **idle** (in `available`, not `busy`), `pop_with(busy, …)` returns `:error` and the destructure `{%{port: port} = client, busy} = …` raises `MatchError` instead of the intended diagnostic — and, raising before `shutdown_clients`, leaks the remaining worker sockets/ports. Idle workers are common early in a build (surplus partition count). Verified. Fix: search `busy ++ available`. (Same file: Low temp-file leak under `$TMPDIR/mix_deps_partition/`, never deleted on any path.)

## 9. `mix xref graph --format stats --label compile-connected` over-counts outgoing deps *(single-file, Medium)*
`lib-mix-tasks-xref.ex:1187` · `out_stats_filter/4`. Iterates out-neighbours `v` but calls `filter_fn.({file, label})`, passing the edge *source* instead of the *target* `v`. The compile-connected check degenerates to "is a compile edge", so any compile edge to a **leaf** file is wrongly counted — the exact command the moduledoc recommends for project health reports files as hotspots the printed graph omits. `in_stats_filter/4` is correct. Verified; fix is `filter_fn.({v, label})`.

## 10. `mix escript.install` on a project with `escript: [path: …]` installs from the wrong location *(cluster, Medium)*
`CLUSTER_lib-mix-tasks-3` · escript.build.ex ↔ escript.install.ex. `escript.build` honors `escript_opts[:path]` and writes the artifact there, but `escript.install`'s `build/2` returns `Mix.Local.name_for(:escripts, …)`, which only consults `:name`/`:app` and ignores `:path`. Result: the escript is built at the configured path but install reads the default location, which doesn't exist → install aborts. Verified.

## 11. `mix escript.install <url>` crashes on a path-less URL *(single-file, Medium)*
`lib-mix-local-installer.ex:128` · `local_install/3`. `URI.parse("http://example.com").path` is `nil`, and `Path.basename(nil)` raises `FunctionClauseError` deep in `IO.chardata_to_string/1` — before any download. escript's permissive `check_install_spec` lets any URL through (unlike archive's `check_extname`). The user gets an opaque crash, not a clear "malformed URL" error. Verified.

## 12. Signed-CSV metadata parser is non-total → opaque crash on Rebar/Hex install *(single-file, Medium)*
`lib-mix-local.ex:219-228` · `find_by_elixir_version/3` (from `find_matching_versions!/3`). Any CSV row that doesn't split into ≥4 fields raises `FunctionClauseError`; any row whose 3rd field isn't a valid version raises `Version.InvalidVersionError`. A truncated download or format drift in the network-supplied CSV aborts the install with an internal stack trace instead of the documented "could not find a matching version" `Mix.raise`. Verified. (Reachability depends on upstream signature verification that this file does *not* perform — see Unresolved ledger, item 19.)

## 13. `create_file(..., format_elixir: true)` crashes on iodata contents *(single-file, Medium)*
`lib-mix-generator.ex:44`. `@spec` allows `iodata` contents, but the `format_elixir` branch passes them straight to `Code.format_string!/2`, which is `is_binary`-guarded → `FunctionClauseError`. The two documented features (iodata + Elixir formatting) are mutually exclusive in practice. Verified; fix normalizes via `IO.iodata_to_binary/1`.

## 14. `:reliable_dir_mtime` default function is always truthy → Windows `.beam` scan never runs *(single-file, Medium)*
`lib-mix-tasks-compile.app.ex:173`. The default is supplied to `Keyword.get/3` as a zero-arity function instead of via `Keyword.get_lazy/3`, so when the key is absent `Keyword.get` returns the *function object* — truthy — and the `cond` always takes the "reliable" branch, never scanning modules, on every OS. On Windows, added/removed `.beam` files go undetected and the generated `.app` `:modules` list goes stale, breaking releases/tooling that read it. Verified.

## 15. Legacy rebar git dep `{git, url, ""}` becomes `ref: ""` instead of `branch: "HEAD"` *(single-file, Medium)*
`lib-mix-rebar.ex:179` · `parse_source/1`. `:file.consult` renders the source string `""` as the empty **charlist** `[]`, never the empty binary the `["" | _]` clause matches, so that clause is dead. The dep gets `ref: ""`, which the Git SCM treats as a real revision and fails to check out — **the dependency can't be fetched**. Verified. (Same file: Low — `git_subdir` emits `:sparse` as a charlist while siblings emit binaries.)

## 16. `mix profile.tprof file.exs <args>` parses/rejects script args the other profilers forward *(single-file + cluster, Medium)*
`lib-mix-tasks-profile.tprof.ex` (`CSCAN-profile.tprof-parse-not-parse-head`) and `CLUSTER_lib-mix-tasks-5`. tprof's `parse!/1` uses greedy `OptionParser.parse!` while cprof/eprof/fprof all use `parse_head!`. So tprof **raises on an unknown `--flag` after the script name** and **consumes known switches meant for the script** — breaking the shared "args after the file go to the executed code" contract the three siblings guarantee. Verified.

## 17. `Mix.Project.push` validates config inside the ProjectStack GenServer, turning a clean error into a fatal crash *(cluster, Medium)*
`CLUSTER_lib-mix-1` · project_stack.ex:263 ↔ project.ex. `push/3` runs `Mix.Project.manifest_path(config)` inside `handle_call`; a malformed user value (e.g. `build_per_environment: "true"` from `mix.exs`, which isn't filtered) makes `manifest_path` `Mix.raise` *inside the max-restarts-0 singleton*, crashing the whole Mix instance with a GenServer crash report instead of the intended one-line error, and every later task fails. A per-file reviewer of either side can't see this. (The `lib-mix-project_stack.ex` per-file report reaches the raise but rules it benign; the cluster view upgrades it.) Confidence medium. Fix: compute the manifest path in the caller's process.

## 18. `Mix.Dep.Lock.write` ignores its own `:file` option *(single-file, Medium — no in-tree trigger)*
`lib-mix-dep-lock.ex:45`. `write/2` compares the new map against the **default** project lockfile (`read()`) instead of the target (`read(lockfile)`), so with the documented `file:` option it can silently skip writing (lost update) or mis-handle `check_locked:`. Verified — but the report's exhaustive caller audit shows **no in-tree production caller passes a divergent `:file`** (mix.ex/fetcher.ex/deps.unlock all resolve to the default), so it's a real public-API contract bug currently exercised only by the test suite. Hence Medium, not High.

---

## Cross-file ledger items to note

- **Contradicted (item 19 — `CROSS_FILE.md`, Medium/medium):** `deps.partition.ex` rules its peer-controlled loopback-TCP `MatchError` and `String.to_unsafe_atom` growth *out of scope*, while the architecturally-identical `sync/pubsub.ex` socket is treated *in-scope* (finding #7) per the folder threat model. `THREAT_MODEL.md`'s "one place with untrusted-binary decoding" invariant overlooked the partition control channel; under the same standard, the partition server's peer-controlled crash/atom-growth are reachable by a local process racing to the listen port. Recommend reconciling the two rulings and the threat model.
- **Unresolved dependencies worth follow-up:** `local.ex` CSV signature verification is assumed but unverified by any caller report (bears on #12's reachability); `tasks/app.config.ex` assumes `config[:config_path]` is never nil; `tasks/loadconfig.ex` CaseClause reproduced only in isolation; `mix.ex` `Mix.install/2` orchestration and `tasks/app.start.ex` `ensure_all_started` error shape were not driven end-to-end.

## Lower-severity findings (Low, verified, worth a cleanup pass)
- **Cluster contract divergences:** `Mix.Shell.Process.print_app/0` returns non-`:ok`, unlike its two siblings and the behaviour callback (`CLUSTER_lib-mix-shell`); `archive.uninstall` moduledoc documents `mix archive.uninstall archive.ez` but installed artifacts are extension-less `APP-VERSION` dirs, so the documented command never matches (`CLUSTER_lib-mix-tasks-0`); `fetcher.ex` hardcodes the `compile.elixir_scm` manifest name instead of reusing `Mix.Dep.ElixirSCM.manifest/1`, an unenforced cross-file invariant (`CLUSTER_lib-mix-dep`).
- **Single-file Lows:** `config.ex` `validate!/2` silently accepts some invalid configs; `scm.ex` non-atomic `Mix.State` read-modify-write and a copy-paste `format_lock` doc; `escript.ex` `File.stat!` crash on a read-only info task; `tasks-deps.deps.tree`/`deps.unlock`/`app.tree`/`archive.build`/`archive.install`/`compile.*`/`local`/`help`/`loadconfig`/`source`/`test.coverage`/`profile.*`/`task.ex`/`utils.ex` each carry minor doc-mismatch, edge-case crash, or cosmetic findings (see the respective per-file reports).

## Files with no surviving findings
`cli.ex`, `dep/converger.ex`, `dep/elixir_scm.ex`, `dep/fetcher.ex`, `dep/loader.ex`, `dep/umbrella.ex`, `dep.ex`, `exceptions.ex`, `hex.ex`, `pubsub.ex`, `pubsub/subscriber.ex`, `remote_converger.ex`, `scm/git.ex`, `scm/path.ex`, `shell.ex`, `shell/io.ex`, `state.ex`, `sync/lock.ex` (one Low, low-confidence non-total decoder note), `project.ex`/`project_stack.ex` (only ruled-out/contract-boundary items), `tasks-server.ex`, `tasks/run.ex`, `tasks/iex.ex`, `tasks/loadpaths.ex`, `tasks/release.init.ex`, `mix.exs` — all reviewed clean or with only ruled-out/out-of-contract items.
