# Cluster cross-file findings

#### `CSCAN-cluster-warnings-as-errors-flag-dropped`

* **Finding ID** — `CSCAN-cluster-warnings-as-errors-flag-dropped`
* **Severity** — Medium
* **Confidence** — high
* **Files involved** —
  * `lib/kernel/cli.ex:16`, `cli.ex:339-341`, `cli.ex:525-536` (produces and forwards the strict-mode flag)
  * `lib/kernel/parallel_compiler.ex:241-243`, `parallel_compiler.ex:309-340`, `parallel_compiler.ex:342-370`, `parallel_compiler.ex:460-488`, `parallel_compiler.ex:849-852` (the intended consumer, which never reads it)
* **Finding shape** — Authz / strict-mode check split (a safety gate is decided in file A and never enforced in file B).
* **Evidence** —
  * `cli.ex:339-341` turns `--warnings-as-errors` into `%{config | warnings_as_errors: true}`, defaulted at `cli.ex:16`.
  * `cli.ex:525-531` builds the compile options `opts = verbose_opts ++ [profile: …, warnings_as_errors: config.warnings_as_errors, return_diagnostics: true]` and `cli.ex:533-536` calls `Kernel.ParallelCompiler.compile_to_path(files, output, opts)`, where the **only** way to force a non-zero CLI exit is `{:error, _, _} -> exit({:shutdown, 1})`. So the whole feature depends on the parallel compiler returning `{:error, …}` when warnings exist and the flag is set.
  * `parallel_compiler.ex:241-243` (`compile_to_path`) forwards `options` verbatim (only injecting `:dest`).
  * The outer `spawn_workers/3` (`parallel_compiler.ex:309-340`) reads only `:max_concurrency` and `:return_diagnostics` from `options`.
  * The inner `spawn_workers/5` (`parallel_compiler.ex:342-370`) copies a *fixed* set of options into the state map (`beam_timestamp`, `dest`, `after_compile`, `each_cycle`, `each_file`, `each_long_compilation`, `each_module`, `profile`, `verification?`, `purge_compiler_modules`) — `:warnings_as_errors` is **not** among them.
  * `verify_modules/4` (`parallel_compiler.ex:460-488`) returns `{:ok, modules, info}` whenever type-check `errors == []`, regardless of how many warnings `info.compile_warnings`/`runtime_warnings` hold; warnings are merely accumulated (`parallel_compiler.ex:849-852`) and never promoted to an error.
  * A repo-wide grep confirms no other consumer: `warnings_as_errors` appears only in `cli.ex`, in `code.ex` (as a *deprecated* `Code.put_compiler_option` whose message says "instead you must pass it as a `--warnings-as-errors` flag"), and in `elixir_erl_compiler.erl:53` where it is *removed* from the erlc option set. The warning sink `elixir_errors.erl:emit_diagnostic/6` has no `warnings_as_errors` gate either.
  * Net effect: `elixirc --warnings-as-errors <files>` compiles with exit status 0 even when the files emit compiler warnings — the strict-mode gate is silently a no-op.
* **What the per-file pass missed** — A `cli.ex`-only reviewer sees the flag correctly parsed and forwarded and reasonably assumes the compiler enforces it; a `parallel_compiler.ex`-only reviewer sees no mention of `warnings_as_errors` at all, so has nothing to flag. Only correlating the producer (`cli.ex`) with the option-plumbing in the consumer (`parallel_compiler.ex`) reveals that the forwarded flag is dropped on the floor.
* **Suggested fix** — Have the parallel compiler honor the option. In the outer `spawn_workers/3` (after obtaining `{status, modules_or_errors, info}` at `parallel_compiler.ex:319-324`), when `Keyword.get(options, :warnings_as_errors, false)` is true and `status == :ok` and `info.compile_warnings ++ info.runtime_warnings != []`, convert the result to `{:error, …, info}` with a synthesized diagnostic (e.g. "Compilation failed due to warnings while using the --warnings-as-errors option") so `cli.ex:535` triggers `exit({:shutdown, 1})`. (If enforcement is instead intended to live outside `Kernel.ParallelCompiler`, `cli.ex` must check `info` warnings itself rather than relying solely on the `{:error, …}` return.)

---

Note on scope: I verified the other inter-file channels in this cluster and found them consistent — the `{:waiting, …}` 9-tuple sent by `error_handler.ex:41-44` matches the receivers at `parallel_compiler.ex:797,802`; `utils.ex:254` `{:available, :struct, module}` matches `parallel_compiler.ex:746`; `:elixir_compiler_dest` written as `{path, force_load?}` at `parallel_compiler.ex:659` matches the `{Dest, Forceload}` reader in `elixir_module.erl:583`; and `handle_deadlock`'s `Map.fetch!(waiting, pid)` is safe because its guard (`map_size(waiting) == map_size(spawned)` with `waiting`-keys ⊆ `spawned`-pids) guarantees every file worker pid is present. The `-e`/`--rpc-eval`/`-S` code-execution paths in `cli.ex` are documented, operator-controlled behavior (out of scope per the folder threat model). No other cross-file security findings surfaced.
