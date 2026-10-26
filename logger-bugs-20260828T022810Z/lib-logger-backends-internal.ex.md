# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/internal.ex
<!-- sha256: 31e6ee8b9acdb301ece8e6f1d41c0903a69d789e8a74dc063eea99f496708513 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:38:32Z_

## Summary
`Logger.Backends.Internal` is a thin, correct supervisor/facade over the legacy backend system; I found no triggerable bugs reachable through its documented public API.

## Bugfixes

No findings.

### Ruled out

- **`add/2` `case` non-total (missing `{:ok, child, info}` clause) → `CaseClauseError`.** The `case` on `Logger.Backends.Supervisor.add/1` (internal.ex:92) only matches `{:ok, _}`, `{:error, {:already_started, _}}`, and `{:error, _}`. I traced the source of the value: `Supervisor.add/1` (supervisor.ex:11) returns whatever `Supervisor.start_child/2` returns for a spec whose start is `Logger.Backends.Watcher.start_link/1`, and `Watcher.start_link/1` (watcher.ex:10) is `GenServer.start_link/2`, which yields only `{:ok, pid}` or `{:error, reason}` — never a 3-tuple `{:ok, pid, info}` (that requires the child's start to return `{:ok, pid, info}`, which `GenServer.start_link` cannot). The `{:error, :already_present}` retry is consumed inside `Supervisor.add/1` and never surfaces. So the reachable shapes are exactly `{:ok, pid}` and `{:error, term}`, both covered. Case is total. (Verified by reading the two callee sources.)

- **`configure/1` last line `:ok = :logger.update_handler_config(...)` raising `MatchError` (internal.ex:40).** `:logger.update_handler_config/3` can return `{:error, term}`, which would break the `:ok =` match. But the `@spec` promises only `:ok`, and under the documented flow `ensure_started/0` has already installed the `Logger` handler, so the refresh returns `:ok`. An error here would require the handler to be absent/broken — not reachable via valid documented use — and is outside the contract's promised return anyway. Not a defect on a valid call path.

- **`configure/1` vs `configure/2` clause ambiguity.** Distinct arities (1 vs 2); no dispatch collision. `configure/2` guards `is_list(options)`; passing a non-list violates the documented `keyword` contract and correctly raises `FunctionClauseError` (out of scope: contract-violating argument).

- **`ensure_started/0` ignoring `Supervisor.start_child` result and always returning `:ok` (internal.ex:122).** Concurrent callers can race, but `start_child` returning `{:error, {:already_started, _}}`/`{:error, :already_present}` is benign (the process is up), and `Process.whereis` short-circuits the common path. A genuine start failure would surface later as a `:gen_event`/handler error, not a silent wrong return from this function. No caller-visible contract is broken.

- **`init/1` calling `:logger.add_handler` before `Supervisor.init` returns (internal.ex:151-159).** This is the same ordering as upstream Elixir; the handler forwards to the `:gen_event` process, and the startup window is transient and unavoidable in the design. Not a triggerable API-level defect.

- **`add/2`/`remove/2` `opts[:flush]` access (internal.ex:90,116).** `opts` defaults to `[]`; Access on a keyword list is valid. Non-list `opts` violates the `add_remove_opts` contract (out of scope).

## Risks / caveats
- This is the upstream Elixir standard-library `Logger` source; the file is small and its correctness leans on `Logger.Backends.Supervisor`, `Watcher`, `Config`, and `Handler`, which are outside this review's single-file scope. My totality/return-shape rulings for `add/2` were established by reading those callees' sources rather than by executing the assembled application (running it requires bringing up the full Logger supervision tree and `:logger` handler), but the return-shape reasoning is deterministic from the code.
- Contract claims about `{:error, :ignore}`/`{:error, reason}` init behavior documented in `add/2`'s `@doc` are actually realized in `Watcher.init/1`, not here; their correctness belongs to that file's review.

