# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/mix_listener.ex
<!-- sha256: 5fdd81fc13b4f98f8e3ff209464e7b001a435239ae38a36bba4f7f9c54756cf9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:10:22Z_

## Summary
`IEx.MixListener` is a small, correct GenServer whose documented public surface (`start_link/1`, `purge/0`) and `:modules_compiled` handling all conform to the Mix compiler-listener protocol, with no triggerable correctness defect found.

## Bugfixes

No findings

### Ruled out

- **`purge/0` whereis→call race returning `exit` instead of `:noop`** — Line 25-26: `purge/0` checks `Process.whereis(@name)` and then does `GenServer.call(@name, :purge, :infinity)`. If the listener died in the window between the two, the `call` would exit rather than return the documented `:noop` ("if the listener is not running"). Rejected as a reportable bug: `MixListener` is a long-lived supervised process, the window is sub-microsecond, it is not attacker-influenced, and this whereis-then-call shape is the idiomatic guard used throughout OTP. It is a theoretical deviation, not a triggerable one; noted under caveats.

- **Type mismatch in `info.os_pid == System.pid()` (line 46) making self-compilation guard always false** — Considered because a string/integer mismatch would defeat the "ignore our own compilations" branch and cause redundant purges. Verified false: the emitter (`Mix.Task.Compiler.notify_modules_compiled`, `/opt/elixir/lib/mix/lib/mix/task.compiler.ex:355`) sets `os_pid: System.pid()`, and `System.pid/0` returns a binary (confirmed by running `is_binary(System.pid()) => true`). Both sides are the same string; the documented contract (task.compiler.ex:70-73) explicitly says to compare with `System.pid/0`. Correct.

- **Missing `:added` in the `modules_diff` match (line 51)** — `%{changed: changed, removed: removed} = info.modules_diff` intentionally ignores `:added` and `:timestamp`. Verified against the protocol doc (task.compiler.ex:63-68): `:added` are brand-new modules with no prior code to purge, so purging only `changed` and `removed` is correct. A partial map match does not raise on the extra keys.

- **Mailbox leak / crash on unexpected messages** — `handle_info/2` has a catch-all clause (line 65-67) returning `{:noreply, state}`, satisfying the protocol's "should have a catch-all clause and ignore other messages" requirement (task.compiler.ex:91-92). No leak.

- **`Enum.empty?`/`Enum.into` misuse on a `MapSet`** — `state.to_purge` is a `MapSet`; `Enum.empty?/1` (line 40) and `Enum.into(list, mapset)` (lines 58-59) both operate correctly on `MapSet` via the `Enumerable`/`Collectable` protocols. `status` computation matches the `@spec purge :: :ok | :noop` and the `@doc`.

## Risks / caveats

- **Runtime toggle of `auto_reload?` can strand already-accumulated modules.** If `IEx.Config.auto_reload?()` is `false` while several compilations accumulate modules into `state.to_purge`, and it is then switched to `true`, the `handle_info` eager-purge branch (lines 53-56) never drains the pre-accumulated set — and in auto-reload mode nothing calls `purge/0`, so those specific stale modules keep their old code until an explicit `purge/0`. This is an edge case requiring a mid-session config change and has minor impact (stale old code lingers, not incorrect behavior); not reported as a defect.
- **`:code.purge/1` in `purge_modules/2` (line 71) hard-purges**, killing any process still executing old code of a `changed`/`removed` module, rather than `:code.soft_purge/1`. This matches Mix's reload semantics and the module's intent, so it is a design choice, not a bug — flagged only for awareness.
- Only the `iex` sub-tree of the Elixir source is present in the working directory; contract claims about the emitter side were verified against the installed toolchain at `/opt/elixir`, which carries a byte-identical copy of this file, so the protocol match is reliable.

