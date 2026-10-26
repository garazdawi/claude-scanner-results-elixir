# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell/quiet.ex
<!-- sha256: 3eb96114b512848a99d6500d7d02cba8d6956131b462c0aefd78041728159970 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell/quiet.ex
<!-- sha256: 3eb96114b512848a99d6500d7d02cba8d6956131b462c0aefd78041728159970 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`mix/shell/quiet.ex` is a thin `Mix.Shell` implementation. Four heads (`print_app/0`, `error/1`, `prompt/1`, `yes?/2`) are `defdelegate`s to `Mix.Shell.IO`; `info/1` is a no-op returning `:ok`; and `cmd/2` runs a command through `Mix.Shell.cmd/3` with a no-op streaming callback. In this re-review I read the two callgraph neighbours the first pass had waved off as "out of scope" — `Mix.Shell.IO` (`shell/io.ex`) and `Mix.Shell` (`shell.ex`) — and confirmed every delegate target, type reference, and spec claim. No genuine, reachable defects were found. Every previously hedged assumption is now verified against the neighbour source; nothing was upgraded to a finding.

## Bugfixes

No findings.

## Risks / caveats

- The file is 66 lines and almost entirely `defdelegate` plus one `cmd/2`. The only real logic (`cmd/2`) was verified behaviourally in the first pass: `Mix.Shell.Quiet.cmd("exit 3")` returns `3` and prints nothing, matching the `@doc` ("Executes the given command quietly without outputting anything") and the `@spec ... :: non_neg_integer`. That evidence stands.

## Did I verify this

- **Delegate targets exist with matching arity (was hedged: "correctness out of scope for this file").** Verified by reading `shell/io.ex`. `Mix.Shell.IO` defines `print_app/0` (io.ex:23), `error/1` (io.ex:42), `prompt/1` (io.ex:52), and `yes?/2` (io.ex:88, with a default arg so `yes?/1` is also generated). Every `defdelegate` in `quiet.ex` (lines 19, 29, 36, 56) resolves to a real function of the correct arity. Not an assumption anymore — read directly. Confidence: high.
- **`Mix.Shell.IO.yes_opts()` type reference in the `@spec` at quiet.ex:55 resolves.** Verified: the type is publicly defined at `shell/io.ex:17` (`@type yes_opts :: [default: :yes | :no]`). Confidence: high.
- **`@spec cmd/2` first-arg tuple form is genuinely accepted.** Verified against `Mix.Shell.cmd/3` (`shell.ex:143-168`): the `command` head pattern-matches `{command, args} -> System.cmd(...)` (shell.ex:163) as well as the binary form (shell.ex:164). So `quiet.ex`'s `cmd(command, opts)` accepting `String.t() | {String.t(), [String.t()]}` is sound — the tuple flows straight through to the accepting callee. Confidence: high.
- **SHELL_CALLBACKS_COMPLETE.** Verified against the `Mix.Shell` behaviour (`shell.ex:39-101`). Required callbacks: `info/1`, `error/1`, `cmd/1`, `cmd/2`, `prompt/1`, `yes?/1`, `yes?/2`, `print_app/0`. Quiet supplies `info/1` (line 24), `error/1` (delegate), `cmd/2` with default opts (lines 63-65, generating `cmd/1`), `prompt/1` (delegate), `yes?/2` with default opts (line 56, generating `yes?/1`), and `print_app/0` (delegate). Full behaviour coverage; no missing callback. Confidence: high.
- **Return-shape / DOCUMENTED_RETURN_SHAPE.** `info/1` returns `:ok`, matching `@callback info(...) :: :ok` (shell.ex:39). Delegated heads inherit `Mix.Shell.IO`'s return values, which I read and match the behaviour's `@callback` shapes (`error :: :ok`, `prompt :: binary`, `yes? :: boolean`, `print_app :: :ok`). Confidence: high.

### Ruled out

- **`cmd/2` callback `fn data -> data end` doesn't print** — not a bug. Intended "quiet" behaviour, matching the module `@moduledoc` and the `cmd` `@doc` ("without outputting anything"). Confirmed against `Mix.Shell.cmd/3`: the callback's return value is explicitly ignored (shell.ex:125 "Its return value is ignored"; the `Collectable` impl at shell.ex:170-178 invokes `fun.(data)` and discards the result), so a no-op callback that neither prints nor mutates cannot affect the returned exit status. Behaviourally verified in first pass (exit status still returned). Axes examined: output side-effect, return value | Not examined: none. Confidence: high.
- **`info/1` returns `:ok` and prints nothing** — matches `@callback info(...) :: :ok` (shell.ex:39) and the documented purpose of the Quiet shell. Intentional. Confidence: high.
- **`@spec cmd/2` (return `non_neg_integer`, tuple-or-string first arg) vs behaviour `@callback cmd/2` (`String.t()`, return `integer`)** — not a reachable defect. The spec is a consistent narrowing/widening: return `non_neg_integer ⊂ integer`, and the wider input tuple form is genuinely accepted by `Mix.Shell.cmd/3` (shell.ex:163). No caller using the documented API is misled; Dialyzer sees a narrower-output/wider-input signature. Axes examined: return-type subset, input-type superset acceptance | Not examined: none. Confidence: high.
- **`@spec yes?` references `Mix.Shell.IO.yes_opts()`** — resolves; type publicly defined at `shell/io.ex:17`. Confidence: high.

### Dropped in re-review

None. Every prior finding and ruled-out entry is preserved; the two "out of scope" hedges were converted to verified entries by reading `shell/io.ex` and `shell.ex`, and no dismissal was strengthened on the basis of an untried axis — the one axis-bearing ruled-out entry (`cmd/2` no-op callback) was already backed by executed behavioural evidence, and reading the callee only confirmed the callback return value is discarded.
