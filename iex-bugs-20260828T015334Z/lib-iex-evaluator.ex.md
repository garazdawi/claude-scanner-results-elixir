# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/evaluator.ex
<!-- sha256: b096603152c4d2131f15df93c7063d9db0e57a7b26130ed83a0b271c2a1443dc -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:07:34Z_

## Summary
After reading the whole module and driving its documented public functions (`parse/3`, `assert_no_error!/0`, `value_from_binding/4`, `variables_from_binding/3`, `fields_from_env/3`) against the built Elixir, I found no triggerable correctness defect reachable through the documented API.

## Bugfixes

No findings

### Ruled out

- **`parse/3` incomplete-vs-error dispatch (`{:error, {_, _, ""}}`)** — I worried the `""` (binary) match on the token slot might miss "incomplete" markers represented as an empty charlist, causing legitimately-continuable input to raise instead of returning `:incomplete`. Verified against the running tokenizer with ten distinct unterminated forms (`foo(`, `if..do`, unterminated string, unterminated heredoc, list, map, `fn ->`, `1 +`, `case..do`, bitstring) — every one returned `:incomplete`. The empty-binary token is the correct representation; no case falls through to `parse_error` wrongly. Verified.

- **`parse/3` `last_op`/`:match` threading (pipe-after-match rejection)** — Checked that the `:match` state carried through an intervening `:incomplete` continuation could spuriously fire the "pipe shorthand not allowed after a match" error. It cannot: the rejection only fires when the *combined* input's first token is an operator, which after a completed match happens only on the immediate next line (the intended case). Verified `x = [1,2]` then `|> Enum.map/2` raises the documented `SyntaxError`, while `x = 1` then `IO.puts(...)` parses cleanly. Verified.

- **`print_error/3` FunctionClauseError branch `pop_in(IEx.inspect_opts()[:syntax_colors][:reset])`** — Suspected a crash-while-printing-an-error (double fault) when `:syntax_colors` is absent from `IEx.inspect_opts()`. Confirmed `IEx.inspect_opts()` default has no `:syntax_colors` key, but `pop_in` over a missing intermediate returns `{nil, opts}` without raising. No crash. Verified.

- **`io_result`/`io_inspect` writing to `:stdio` directly** (GROUP_LEADER_IO_ONLY concern) — `IO.puts(:stdio, …)`/`IO.write(:stdio, …)` map `:stdio` to the calling process's group leader, which `init/5` sets via `Process.group_leader/2`. Output goes through the group leader as the contract requires. Ruled out on the documented `IO` device-mapping mechanism.

- **`init/5` re-entrant `send(self(), {:done, old_evaluator, false})` not matched by the nested `loop/1`** — The nested loop matches only `^ref`/`^server`; `old_evaluator` is the *outer* loop's ref. This is intentional cross-level signaling: the message stays in the mailbox until the outer `loop/1` resumes (its `state.ref == old_evaluator`) and consumes it via `{:done, ^ref, next?}` to terminate the superseded evaluator during takeover. Not a leak.

- **`loop/1` rebinding `ref` in the `:*_from_binding`/`:fields_from_env` clauses** — In a `receive` pattern an unpinned already-bound name shadows/rebinds rather than pins, so these clauses correctly capture the caller's request ref and reply to it; the state ref is used only where pinned (`^ref` in the two `:done` clauses). Correct by design.

- **`traverse_binding`/`value_from_binding` on non-map values and dead targets** — Traversal of a nested map returns `{:ok, value}`, a non-map value under a key path returns `:error`, and an unresponsive/dead evaluator returns `:error` after the 5s timeout — all matching the `{:ok, any} | :error` spec. Verified.

- **`prune_stacktrace/1` comparing to exact atom `:elixir_compiler`** — In the eval path no frame carries that module, so `drop_while` empties the list and the explicit `[] -> stack` fallback returns the full stacktrace. This is intended best-effort pruning (trim only when a compiler frame is present, as for `c/1`-compiled code), not a defect.

## Risks / caveats
- `init/5` carries a `@doc` (attached to the module eval-loop description) but is realistically only invoked by `IEx.Server` via `:proc_lib`; I treated its message protocol as internal and did not attempt to drive it with adversarial arguments, since that requires violating its undocumented calling contract (out of scope).
- Multi-line/continuation behavior of `parse/3` depends on the exact error shapes returned by `:elixir.string_to_tokens`/`tokens_to_quoted` in this OTP-29 / Elixir 1.21.0-dev build; findings were confirmed against that build and could differ on other tokenizer versions, though the dispatch logic is defensive (fallback to `:incomplete` only on the empty-token marker).
- I did not exercise the `.iex.exs` loading path (`load_dot_iex`/`eval_dot_iex`) end-to-end; its `catch`-all correctly prints and returns the prior `state`, and the trust boundary (a file evaluated from cwd) is a documented, operator-accepted risk rather than a code defect.

