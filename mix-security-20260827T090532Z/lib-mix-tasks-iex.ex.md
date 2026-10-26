# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/iex.ex
<!-- sha256: ce802496ee88efb6a39b560d9f0d5bf3e542fa82413008093dcb5d5615d4a382 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/iex.ex
<!-- sha256: ce802496ee88efb6a39b560d9f0d5bf3e542fa82413008093dcb5d5615d4a382 -->

_Re-reviewed on 2026-08-27_

The module `Mix.Tasks.Iex` (16 lines) defines a single public entry point `run/1`. It:
- discards its argument entirely (`_`),
- calls `Mix.raise/1` with a **static string literal** — no interpolation of any external value.

## Summary
`Mix.Tasks.Iex.run/1` ignores its argument and raises a fixed instructional string; it contains no data-flow sink and no attacker-reachable behavior.

## Security
No findings

### Ruled out
- **Template / interpolation in `Mix.raise` (line 14)** — the message is a static literal (`"To use IEx with Mix, please run \"iex -S mix\""`) with no `#{}` interpolation; the escaped quotes are literal characters, not external data. I read the callee: `Mix.raise/1` (mix.ex:623) delegates to `Mix.raise/2` (mix.ex:637), which does `Kernel.raise(Mix.Error, mix: status, message: message)` — the binary is stored on the exception struct and passed through verbatim, with no format-directive interpretation, no `IO.format`-style parsing, and no evaluation. No externally-derived value reaches the raise, and the raise itself introduces no new sink. `Axes examined: interpolation of external data into message, format-string interpretation by callee | Not examined: none.`
- **Argument handling in `run/1` (line 13)** — the task argv is bound to `_` and never used, so no external input flows anywhere. This is a syntactic fact of the file, fully verified by reading it. `Axes examined: whether argv reaches any operation | Not examined: none.`
- **`use Mix.Task` behaviour (line 6)** — standard Mix behaviour injection. `Axes examined: presence of macro-level evaluation of external input at this call site | Not examined: none — no external input exists in this file for the behaviour to act on.`

### Did I verify this
- I read the target file in full (16 lines) and confirmed `run/1` matches its argument with `_` and calls `Mix.raise` with a compile-time constant.
- I read the callgraph neighbour `Mix.raise/1` and `Mix.raise/2` in `lib/mix/lib/mix.ex:623-648` and confirmed the message binary is forwarded unmodified to `Kernel.raise(Mix.Error, ...)` — no format interpretation, no interpolation, no dispatch on the string contents.
- No reproduction was runnable because there is no data-flow sink and no external input reaches any operation.

### Dropped in re-review
None. All prior ruled-out entries are preserved; the sole first-pass hedge ("Any actual IEx-under-Mix behavior lives in `iex -S mix` ... outside this file") was a scope note, not a sink assumption — the security-relevant claim it depended on (that `Mix.raise` does not itself constitute a sink for the static message) is now verified by reading `mix.ex`, so no upgrade to a low-confidence finding is warranted.
