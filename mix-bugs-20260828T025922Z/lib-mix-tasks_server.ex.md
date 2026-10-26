# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks_server.ex
<!-- sha256: 91d9b231bc3b1ce3614de44bbe2b63182f00290b9b5087e172e677d2019278c3 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks_server.ex
<!-- sha256: 91d9b231bc3b1ce3614de44bbe2b63182f00290b9b5087e172e677d2019278c3 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`Mix.TasksServer` is a tiny internal (`@moduledoc false`) Agent wrapping a named public ETS `:set` table; its five functions are total and use a consistent key (`tuple`) across insert/member/delete. On re-review I verified the two cross-file assumptions the first pass left hedged by reading the actual callers (`Mix.Task` in `task.ex`) and the supervision start (`mix.ex`), and both hold. I found no reachable behavioural defect.

## Bugfixes

No findings.

The file is small enough to reason about exhaustively:

- `run/1` inserts `{tuple}` with `:ets.insert_new/2`, giving the intended "run at most once" semantics (returns `false` when already present).
- `put/1` uses `:ets.insert/2` and returns `:ok` unconditionally — matches an idempotent set-membership put.
- `get/1` uses `:ets.member(@name, tuple)`, and `delete_many/1` uses `:ets.delete(@name, &1)`. Both key on `tuple` directly, which is exactly the key of an object stored as `{tuple}` under the default `keypos` of 1.
- `clear/0` uses `:ets.delete_all_objects/1` and returns `:ok`.

### Key consistency — verified against callers (was hedged "outside this file")

The first pass verified key consistency only empirically against a synthetic key. I have now read every caller in `lib/mix/lib/mix/task.ex` and confirmed the keys are structurally consistent across the write/read/delete surface:

- Writes: `Mix.TasksServer.run({:alias, task, proj})` (task.ex:425, 456, 497), `Mix.TasksServer.put({:task, original_task, proj})` (task.ex:604) — stored as `{ {:alias|:task, task, proj} }`.
- Reads: `Mix.TasksServer.get({:task, task, proj})` (task.ex:428) — `:ets.member` keyed on the same 3-tuple.
- Deletes: `Mix.TasksServer.delete_many([{:task, task, proj}, {:alias, task, proj}])` (task.ex:636, 642, 648) — `many` is a list of those exact 3-tuples, each passed to `:ets.delete(@name, &1)`, matching the stored key.

So `run({:task,…})` → `get({:task,…})` → `delete_many([{:task,…},{:alias,…}])` all agree on the key shape. The empirical run from the first pass still stands as corroboration:

```
$ elixir -e ':ets.new(:t, [:public, :set, :named_table]); IO.inspect(:ets.insert_new(:t, {{:a, :b}})); IO.inspect(:ets.member(:t, {:a, :b})); IO.inspect(:ets.insert_new(:t, {{:a, :b}})); :ets.delete(:t, {:a, :b}); IO.inspect(:ets.member(:t, {:a, :b}))'
true
true
false
false
```

Insert reports `true`, membership `true`, a second insert reports `false`, delete removes the key — exactly what the `task.ex` `cond` branches rely on.

## Risks / caveats

- The table is created `:public`, so readers (`get`, and the ETS access from other Mix modules) bypass the owning Agent and can race with writers under concurrency. This is a deliberate design choice documented in the folder threat model ("readers use raw `:ets` … bypassing the owning process") rather than a defect in this file, so I did not raise it as a finding.
- No `handle_info`/message-handling concerns apply — this is a plain `Agent` with no custom callbacks, so the "GenServer must tolerate unknown messages" obligation is satisfied by the `Agent` default.

## Ruled out

- **Table-not-created precondition (`ArgumentError` before `start_link`)** — the first pass hedged this as "outside this file's control." Verified: `mix.ex:464` places `Mix.TasksServer` in the `Mix.Supervisor` children list started by `Mix.start/0` (via `Application.ensure_all_started(:mix)`, `mix.ex:455-467`), before any task dispatch occurs. All five functions would raise `ArgumentError` from `:ets` only if called before `start_link/1`, which the supervision tree precludes for the normal Mix run. This is a genuine contract precondition satisfied by the boot sequence, not a bug in the documented API surface.\
  Axes examined: caller key shape (task.ex), table-creation ordering (mix.ex supervision), ETS insert/member/delete semantics (executed) | Not examined: none.

## Did I verify this?

- Read `tasks_server.ex` in full (39 lines).
- Read `mix.ex:455-474` — confirmed `Mix.TasksServer` is a supervised child, resolving the "table created before use" hedge.
- Read `task.ex:415-464, 497, 604, 636-648` and grepped all `TasksServer` references — confirmed every caller uses the `{:task|:alias, task, proj}` 3-tuple as key across run/get/put/delete_many, resolving the "key consistency outside this file" hedge.
- Read `state.ex` (sibling singleton) for corroboration of the public-ETS pattern; no bearing on findings here.
- Re-ran the ETS insert_new/member/delete sequence in the sandbox toolchain; behaviour matches caller expectations.

## Dropped in re-review

No prior findings were dropped — the first pass reported none. The two hedged cross-file assumptions ("outside this file") were both **verified** (not upgraded to findings) by reading `task.ex` and `mix.ex`, since executed/read evidence confirms the invariants hold rather than contradicting them.
