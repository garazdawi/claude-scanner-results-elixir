# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/project_stack.ex
<!-- sha256: ff38977ff7ce1e6607d8b7fceb69dcfa2b130b466b387cc2fea29315a5f16151 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:20:12Z_

## Summary

`Mix.ProjectStack` is an internal (`@moduledoc false`) GenServer singleton holding the project stack and post-config; the code is total over the states its documented callers actually produce, and I found no reachable behavioural defect through its normal call flow.

## Bugfixes

No findings

## Risks / caveats

- The entire module is `@moduledoc false`; there is no externally documented public API. I treated the `@spec`s and the invariants established by the sole in-tree caller (`Mix.Task`) as the contract. Every `@spec` I checked (`config_mtime :: integer`, `reset_config_mtime :: binary`, `pop_post_config :: term`, `pop_after_compiler :: [fun]`, `push :: :ok | {:error, file}`) matches the implementation's return shape.
- I ran the empty-stack read paths and confirmed they return sane values (`config_mtime` → `946684800`, `peek`/`pop`/`top_and_bottom` → `nil`), and I reproduced the two non-total functions below to confirm they are guarded by their caller rather than reachable in normal operation.
- I did not attempt to construct concurrent/racing callers (the module serializes all mutation through one GenServer, but readers like `get_stack` interleaved with `update_stack` across two separate `GenServer.call`s in `config_mtime` are theoretically race-prone). Under Mix's single-threaded task-driving model this is not reachable, so I did not treat it as a finding.

### Ruled out

- **`on_recursing_root/1` MatchError when no project is recursing (line 68)** — `Enum.split_while(stack, &(not &1.recursing?))` returns `{stack, []}` when no element has `recursing?: true`, and `{top, [mid | bottom]} = ...` then raises `MatchError`. *Verified* by driving `push/3` followed by `on_recursing_root/1` with no recursing project (`elixir /tmp/t2.exs` produced the `MatchError` at `project_stack.ex:68`). However, the only caller (`Mix.Task` line 495) invokes it exclusively inside `not recursive && Mix.ProjectStack.recursing() ->`, i.e. only when `recursing()` is non-nil — so a recursing element always exists. Not reachable through normal flow; the crash requires violating the caller-side invariant.
- **`push/3` crashes the ProjectStack GenServer on invalid config (line 263)** — `Mix.Project.manifest_path(config)` runs inside `handle_call`, and an invalid config value (e.g. `build_per_environment: "yes"`) makes it `Mix.raise`, terminating the singleton. *Verified* the raise reproduces (`/tmp/t.exs`). Ruled out as a behavioural bug: the same `Mix.Error` is surfaced to the caller via the `GenServer.call` exit and Mix aborts regardless; `manifest_path` performs this validation identically wherever it runs, so no wrong value or silent failure results — only the process that was about to abort dies.
- **`config_mtime/0` `Enum.max([])` on empty `config_files`** — would raise `Enum.EmptyError`, but `config_files` is seeded in `push/3` as `[manifest_file | parent_files]` (always ≥1 element) and only ever grown by `loaded_config/2` (`files ++ h_files`), never emptied. Empty case unreachable.
- **`config_mtime/0` second `update_stack` clause `[h | t]` non-total** — only reached when the first `get_stack` returned a list, which only happens for a non-empty stack head (`config_mtime: nil`); single-threaded flow keeps the stack non-empty between the two calls.
- **`clear_stack/0` also clearing `post_config`** — `update_state(fn _ -> {:ok, initial_state()} end)` resets both stack and post_config despite the name. Intentional full reset used on clean-slate paths; no incorrect observable behaviour for callers.
- **`printable_app_name/0` resetting all tail entries to `io_done: false`** — matches the documented shell-ownership behaviour described in the `push/3` comment; deliberate, not a defect.

