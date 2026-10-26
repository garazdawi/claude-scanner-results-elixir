# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell/process.ex
<!-- sha256: f1660332f79a6fb08a64bcd6f4b8868196b259ce8825e04ba93bd3b0071bc2ba -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:26:47Z_

## Summary

`Mix.Shell.Process` is a small, mostly-correct process-mailbox shell used for testing; the one genuine defect is that its `print_app/0` violates the `@callback print_app() :: :ok` contract by returning `nil` or a message tuple instead of `:ok`.

## Bugfixes

#### `CSCAN-process-print-app-not-ok`
- **Finding ID** — `CSCAN-process-print-app-not-ok`
- **Severity** — Low
- **Confidence** — high
- **Location** — `elixir/lib/mix/lib/mix/shell/process.ex:77-81`, function `print_app/0`
- **Class** — Contract violation (callback return shape)
- **Impact** — `Mix.Shell` declares `@callback print_app() :: :ok`. This implementation returns the value of its `if` expression: `nil` when there is no printable app name, or the result of `send/2` (the tuple `{:mix_shell, :info, ["==> name"]}`) when there is. It never returns `:ok`. A caller of the documented `Mix.shell().print_app()` that relies on the documented `:ok` return gets a wrong value, and Dialyzer sees a callback-conformance mismatch that propagates into every analysis consuming this behaviour implementation. The sibling `Mix.Shell.IO.print_app/0` (`io.ex:23-29`) explicitly returns `:ok`, confirming which side is correct.
- **Evidence** — `verified`.
  - Command: `elixir /tmp/t.exs` driving `Mix.Shell.Process.print_app()` with no project entered.
  - Output: `print_app (no app) returns: nil` / `== :ok: false`, while `IO.print_app returns: :ok`.
  - Command: `elixir -e 'IO.inspect(send(self(), {:mix_shell, :info, ["==> foo"]}))'` → `{:mix_shell, :info, ["==> foo"]}`, confirming the app-present branch returns the message tuple rather than `:ok`.
- **Suggested fix** — Add a trailing `:ok` after the `if`, mirroring `Mix.Shell.IO.print_app/0`:
  ```elixir
  def print_app do
    if name = Mix.Shell.printable_app_name() do
      send(message_target(), {:mix_shell, :info, ["==> #{name}"]})
    end

    :ok
  end
  ```

## Risks / caveats

- I could not construct a live project state where `printable_app_name/0` returns a non-nil atom without a full Mix project, so the app-present return value was established by composing the verified `nil`-branch behaviour with the verified return value of `send/2` rather than by a single end-to-end run; both sub-facts are `verified`.
- I considered the asymmetry whereby `info`/`error`/`prompt` send output to `message_target/0` (the `$callers` parent, when set) while `flush/1`, `prompt/1`, and `yes?/2` read from `self()`'s mailbox. I ruled it out below as intended design rather than a defect, but note it rests on the `$callers` forwarding convention being deliberate; if that convention were ever expected to symmetrically redirect the input/flush side too, this would be worth revisiting.

### Ruled out
- **`prompt/1` / `yes?/2` `receive ... after 0` raising** — The `@doc` explicitly documents that the input message must already be in the mailbox and that the call aborts otherwise; raising on absent input is the stated contract, not a bug.
- **`message_target/0` vs `flush/self()` mailbox asymmetry** — Output is forwarded to the `$callers` parent so that spawned tasks' shell messages reach the test process, while input/flush operate on the local mailbox. This matches how Mix forwards shell messages up the caller chain and how the documented examples (all same-process, using `self()`) work; no documented public call path produces a wrong result from it.
- **`yes?/2` returning a non-boolean** — It returns whatever `{:mix_shell_input, :yes?, response}` carries; the `@doc` states `value must be true or false`, so a non-boolean requires a contract-violating input from the caller and is out of scope.
- **`cmd/2` return value** — Delegates to `Mix.Shell.cmd/3`, which returns the integer exit status, matching `@callback cmd(...) :: integer`.

