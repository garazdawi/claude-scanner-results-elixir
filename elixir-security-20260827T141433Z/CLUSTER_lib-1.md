## Investigation summary

I mapped every cross-file edge among the 8 cluster files and checked each against the security lens:

- **config.ex → File.read! → Code.eval_string** (config.ex:314, 365): the config-evaluation path. Config content is author/operator-controlled and evaluating it is the module's documented purpose; the per-file passes for both config.ex and code.ex traced this end-to-end (reading `Config.Reader`/`Config.Provider` and `src/elixir.erl`) and confirmed it is by-design, out of scope per the folder threat model. No cluster caller feeds *untrusted* content into it.
- **code.ex → File.read / File.read!** (code.ex:1145 `format_file!`, 2373/2408 docs & `find_file!`): by-design file reads for formatting/eval; null bytes are rejected by OTP `:file` (verified in the file.ex per-file pass), so no null-byte or path-validation split reaches a sink.
- **enum.ex → Collectable / Dict** (enum.ex:1080/1445, collectable protocol): by-design protocol dispatch; both sides verified in their per-file passes.
- **Dangerous sinks** (`binary_to_term` code.ex:2383; `apply` exception.ex:2690, code.ex:656, dynamic_supervisor.ex:766): none receives argument taint originating in a *different* cluster file. In particular, the two real per-file findings do not chain across the cluster — no cluster file calls `Code.fetch_docs` on an untrusted `.beam`, and no cluster file calls `Exception.normalize/blame/format` with an untrusted stacktrace (grep confirmed zero such call sites).
- **Shared mutable state**: the process-dictionary keys are disjoint across files (`:code_formatter_comments`, `{Config, …}` tuples, `:"$initial_call"`); no cross-file corruption.
- **Exception def/use splits** (`File.Error`/`File.CopyError`, `Enum.OutOfBoundsError`/`EmptyError`, `Code.LoadError` all defined in exception.ex, raised in file.ex/enum.ex/code.ex): the standard core-exception arrangement; every `message/1` only formats via `inspect` (default `limit: 50`) or `:file.format_error/1` on trusted `:file` reasons — no injection, unbounded amplification, or crash on the fields the raising files populate.

None of these seams re-opens a hole that a single-file reviewer closed, crosses a trust boundary the per-file passes hadn't already adjudicated, or crashes on attacker-reachable input. The existing `CROSS_FILE.md`/`CLUSTER_root-*` findings all concern the Erlang `src/` compiler files, not these stdlib modules.

# Cluster cross-file findings

No cluster cross-file findings
