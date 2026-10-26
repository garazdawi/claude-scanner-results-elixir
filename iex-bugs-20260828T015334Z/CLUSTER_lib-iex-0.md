# Cluster cross-file findings

#### `CSCAN-cluster-broker-evaluator-nil-pid`

- **Finding ID** — `CSCAN-cluster-broker-evaluator-nil-pid`
- **Severity** — Medium
- **Confidence** — medium
- **Files involved**
  - `broker.ex:33-42` (`evaluator/1` definition and `@spec`)
  - `autocomplete.ex:830-863` (the four `*_from_env` / `*_from_binding` bridge helpers)
  - `evaluator.ex:165-205` (`value_from_binding/4`, `variables_from_binding/3`, `fields_from_env/3` — each does `send(evaluator, …)`)
- **Finding shape** — Inherited assumption broken / type-contract mismatch across files.
- **Evidence**
  - `broker.ex:36` declares `@spec evaluator(shell()) :: {evaluator :: pid, server :: pid} | nil`, but the body (`broker.ex:37-42`) unconditionally returns `{dictionary[:evaluator], pid}`. When the shell pid exists but has no `:evaluator` key in its process dictionary, `dictionary[:evaluator]` is `nil`, so the function returns `{nil, pid}` — a 2-tuple whose first element is **not** a pid, violating the spec. (Additionally, if `pid` is a dead process, `Process.info(pid, :dictionary)` returns `nil` and the `{:dictionary, dictionary} = …` match on `broker.ex:39` raises `MatchError` rather than returning `nil`.)
  - The `:evaluator` key is only transiently present: `server.ex:113` sets it and `server.ex:120` (`stop_evaluator`) deletes it during a rerun/respawn before `start_evaluator` re-sets it, leaving a window where the shell pid has no `:evaluator`.
  - Every autocomplete bridge uses `with {evaluator, server} <- IEx.Broker.evaluator(shell)` (`autocomplete.ex:831, 841, 850, 858`). A `{nil, pid}` result still matches `{evaluator, server}`, so `evaluator` binds to `nil` and control does **not** fall to the `else -> []`/`:error` clause that only triggers on a plain `nil` return.
  - `evaluator` is then passed straight into `IEx.Evaluator.fields_from_env(nil, server, …)` / `variables_from_binding(nil, …)` / `value_from_binding(nil, …)`, whose first action is `send(evaluator, …)` (`evaluator.ex:167, 183, 198`). `send(nil, msg)` raises `ArgumentError`, which propagates out of the non-`remsh` `expand/2` path (`autocomplete.ex:65-70`), which has no surrounding `try`.
- **What the per-file pass missed** — A reviewer of `broker.ex` alone sees a spec of `{pid, pid} | nil` and no obvious problem; a reviewer of `autocomplete.ex` alone trusts that spec and treats the result as "tuple ⇒ live evaluator pid." Only by reading both together (plus the `send/2` sites in `evaluator.ex`) does the `{nil, pid} → send(nil, …)` crash become visible.
- **Suggested fix** — Make `broker.ex` `evaluator/1` honor its contract: guard the dictionary read and return `nil` when the pid is dead or `dictionary[:evaluator]` is `nil`, e.g.

  ```elixir
  def evaluator(pid \\ shell()) do
    with pid when is_pid(pid) <- pid,
         {:dictionary, dictionary} <- Process.info(pid, :dictionary),
         evaluator when is_pid(evaluator) <- dictionary[:evaluator] do
      {evaluator, pid}
    else
      _ -> nil
    end
  end
  ```

  so the existing `with {evaluator, server} <- …` clauses in `autocomplete.ex` correctly fall through to their empty-result branches.
