# /Users/lukas/code/claude-scanner/elixir/lib/mix/mix.exs
<!-- sha256: d032569d7714a3044dbbcbe9d7658972194f07f57a1afff34db1b50b8f1de5e9 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/mix.exs
<!-- sha256: d032569d7714a3044dbbcbe9d7658972194f07f57a1afff34db1b50b8f1de5e9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`elixir/lib/mix/mix.exs` is Mix's own project-definition module: two zero-arity callbacks (`project/0`, `application/0`) returning literal keyword lists. There is no branching, pattern matching, external input, or resource acquisition, so none of the runtime failure classes in the inventory have a site here. The one substantive question — whether the `:registered` application key matches the set of processes Mix actually registers — I resolved this pass by reading the supervisor tree and each child's `start_link/1`. It does **not** match: `Mix.Sync.PubSub` is a registered singleton that is missing from the list. I record that as a single low-confidence finding (upgraded from a hedged caveat in the first pass). The second first-pass hedge (build-time resolution of `System.version/0` / `Mix.CLI`) is now verified and cleanly ruled out.

## Findings

### 1. `:registered` list omits the `Mix.Sync.PubSub` singleton — **Upgraded from hedged assumption in first pass**

- **Location:** `mix.exs:19` — `registered: [Mix.State, Mix.TasksServer, Mix.ProjectStack]`
- **Confidence:** Low
- **Sink class:** Documented-contract / application-metadata correctness (soundness of the `:registered` application key; not a runtime crash).

The Mix application supervisor starts four named children (`lib/mix/lib/mix.ex:464`):

```
children = [Mix.Sync.PubSub, Mix.State, Mix.TasksServer, Mix.ProjectStack]
```

I confirmed each of these registers a local name equal to its module:

- `Mix.State` — `GenServer.start_link(__MODULE__, :ok, name: @name)` with `@name __MODULE__` (`state.ex:7,13`)
- `Mix.TasksServer` — `Agent.start_link(…, name: @name)` (`tasks_server.ex:11-14`)
- `Mix.ProjectStack` — `GenServer.start_link(__MODULE__, :ok, name: @name)` (`project_stack.ex:21-22`)
- `Mix.Sync.PubSub` — `GenServer.start_link(__MODULE__, {}, name: @name)` with `@name __MODULE__` (`sync/pubsub.ex:33,36-37`)

The `:registered` list enumerates the first three but omits `Mix.Sync.PubSub`, which is registered under the name `Mix.Sync.PubSub` and lives in the same supervisor tree. `:registered` is advisory metadata consumed by release-conflict detection; an omitted name means a name clash on `Mix.Sync.PubSub` would not be caught by that tooling. It does **not** break `application/0`'s contract (the app still boots; the callback still returns a valid spec), so severity is low — but the list is genuinely incomplete against the observed registrations, so this is no longer a "did-not-check" caveat but a verified discrepancy. Most likely `Mix.Sync.PubSub` was added to the supervisor after the `:registered` list was last updated.

## Ruled out

- **`System.version/0` / `Mix.CLI` fail to resolve at build time** (first-pass hedge #2).\
  Verified. `System.version/0` is an Elixir stdlib function (always present). `Mix.CLI` is defined at `lib/mix/lib/mix/cli.ex` (confirmed via grep for `defmodule Mix.CLI`). `Mix.State`, `Mix.TasksServer`, `Mix.ProjectStack`, and `Mix` all exist in-tree and are the correct singletons/entry points. The `escript`/`version`/`mod` entries therefore cannot fail to evaluate.\
  *Axes examined:* symbol existence of every referenced module and function. *Not examined:* none — every referenced symbol was located in-tree.

- **Runtime failure classes (state-threading, silent-swallow, binary-decode crash, resource leak, atom exhaustion, `cond`/`case` fallthrough).**\
  No site: the file contains only literal keyword lists in two zero-arity functions — no input, no branching, no side effects.\
  *Axes examined:* control flow, external input, resource acquisition. *Not examined:* none.

## Did I verify this?

Yes for both first-pass hedges.

- Hedge #1 (registered-list completeness) — I read `lib/mix/lib/mix.ex` (supervisor children, line 464-466) and each child module's `start_link/1` (`state.ex`, `tasks_server.ex`, `project_stack.ex`, `sync/pubsub.ex`) to confirm the actual set of registered names, and compared it to the `:registered` literal. The comparison shows `Mix.Sync.PubSub` missing. This is a static-fact comparison (which names are registered vs. which are listed), fully answerable by reading — no execution axis is involved — so reading is sufficient here and the item is upgraded to a low-confidence finding rather than left asserted.
- Hedge #2 (build-time symbol resolution) — verified by locating each referenced module/function in the source tree.

## Dropped in re-review

None. The first pass reported no findings; the sole prior caveat has been upgraded (not dropped) to Finding #1, and the second caveat has been verified into the Ruled-out section.
