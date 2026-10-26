# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex.ex
<!-- sha256: cdc696618f1d653131e20912907f61a4c6722e42c0208422d1f8f97c43811d43 -->

# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex.ex
<!-- sha256: cdc696618f1d653131e20912907f61a4c6722e42c0208422d1f8f97c43811d43 -->

_Re-reviewed on 2026-08-27_

Full coverage of the file (938 lines, all definitions read). This re-review resolves the first pass's "single-file review" hedge by reading the two delegation neighbours whose behaviour the ruled-out entries depended on (`IEx.Config`, `IEx.Pry`) and by re-running the one behavioral claim (`color/2` escape passthrough). No assumption survived that needed upgrading to a finding.

## Summary

`iex.ex` is the documentation-heavy public-API facade for `IEx`. Every security-relevant sink is a thin delegation (`IEx.Config`, `IEx.Pry`) or operates on developer-controlled AST/config. I verified the two delegation targets rather than assuming them: `IEx.Config.color/1` returns only `nil` or a developer-configured value, and `IEx.Pry.break!/4`/`break!/6` exist with the signatures the facade calls. No attacker-reachable defect exists in this file.

## Security

No findings

### Ruled out

- **`color/2` ANSI/terminal-escape passthrough (line 633)** — `[ansi | string] |> IO.ANSI.format(true) |> IO.iodata_to_binary()` emits `string` verbatim.
  - *Verified (executed):* re-ran the exact pipeline with `string = "abc\e]0;pwn\a"`, `ansi = [:red]`. Output bytes `<<27,91,51,49,109, 97,98,99, 27,93,48,59,112,119,110,7, 27,91,48,109>>` — the raw OSC `\e]0;pwn\a` survives between the color-set and reset sequences (`String.contains?/2` → `true`).
  - *Verified (neighbour read):* `IEx.Config.color/1` (config.ex:77–90) returns `nil` when coloring is disabled, else `Keyword.fetch`/`default_color` — a value drawn from `Application.get_env(:iex, :colors, [])` (operator/developer config), never from untrusted network/disk data. So neither argument to `color/2` originates from an attacker within this file: `color` and `string` are both caller-supplied, and the `ansi` value is developer config.
  - Terminal-injection risk therefore lives only at call sites in *other* files that pass externally-derived text into `color/2`; the wrapper itself introduces no boundary. Confidence: High (executed + neighbour-confirmed).
- **`configure/1`, `configuration/0`, `started?/0`, `width/0`, `inspect_opts/0`, `after_spawn/*` (lines 592–658)** — one-line delegations to `IEx.Config`.
  - *Verified (neighbour read):* `configure/1` → `Agent.update(..., handle_configure, ...)` which runs `validate_option/1` on each option (config.ex:180–212) and only writes keys in `@keys` via `Application.put_env(:iex, key, value)`. The dangerous keys (`:dot_iex` → file eval, `:parser` → `{m,f,a}` dispatch) are validated for *shape* only (`is_binary`, `tuple_size == 3`) and their evaluation sinks live in `IEx.Server`/`IEx.Evaluator`, not here. All writes are to the `:iex` app-env namespace (operator/local config), no cross-tenant reach. Confidence: High.
- **`pry/0` macro (line 754)** — expands to `IEx.Pry.pry(binding(), __ENV__)`; pure hygienic `quote`, no external input. Confidence: High.
- **`break!/2` macro + `__break__!/3,/7` (lines 764–800)** — `Macro.escape/1`, `Macro.decompose_call/1`, and `Macro.expand(module, env)` operate on the AST the developer literally typed as `break! Mod.fun/arity`, expanded in the shell's own `__ENV__`. `Macro.expand` can run macros, but only ones the developer referenced in a context where they already hold full code-execution; no trust boundary is crossed. `__break__!` is a `__name__`-convention helper invoked only by the `break!` macro. Confidence: High.
- **`arity in 0..255` guard (line 770)** — reached on every `break! Mod.fun/N` form; non-integer/out-of-range arity falls through to the catch-all `__break__!/3` clause (line 788) → `raise_unknown_break_ast!/1`. Reachable and correct, not dead defensive code. Confidence: High.
- **`raise_unknown_break_ast!/1` `Macro.to_string(ast)` (line 810)** — AST-to-string only for an `ArgumentError` message; developer-supplied AST, result not re-evaluated. Confidence: High.
- **`break!/4` defdelegate (line 933), `dont_display_result/0` (line 936)** — delegate to `IEx.Pry` / return a literal atom.
  - *Verified (neighbour read):* `IEx.Pry.break!/4` (pry.ex:237) and `IEx.Pry.break!/6` (pry.ex:245) both exist with the arities the facade's `__break__!` helpers and the `defdelegate` call. `dont_display_result/0` returns the fixed literal atom `:"do not show this result in output"` — no dynamic atom creation. Confidence: High.

## Risks / caveats

- The genuinely dangerous IEx surfaces — `.iex.exs`/`import_file` evaluation, `IEx.Broker` session takeover and the `Allow? [Yn]` pry-consent gate, `remsh`/`erpc` cross-node fan-out, and atom creation in autocomplete/history — all live in sibling files (`iex/config.ex` beyond the read functions, `iex/server.ex`, `iex/broker.ex`, `iex/pry.ex`, `iex/evaluator.ex`, `iex/autocomplete.ex`). This facade is out of the blast radius for those; review them separately. Note: `IEx.Config.parser/1` (config.ex:71–75) does default the parser to `{IEx.Evaluator, :parse, []}` and can be overridden by app-env, but that dispatch fires in the evaluator, not in `iex.ex`.
- Consistent with the folder threat model, IEx's evaluator input is trusted-by-design (arbitrary code execution is the feature); the real defended boundary is who can reach the group leader / distribution port — neither implemented in this file.

## Did I verify this?

- **`color/2` passthrough** — Yes, executed. Re-ran the pipeline; OSC escape survives verbatim (bytes cited above). Confirmed the input `ansi` value comes only from developer config by reading `IEx.Config.color/1`.
- **`configure/1` delegation** — Yes, neighbour-read. `handle_configure` validates option shapes and writes only `@keys` to `:iex` app-env; no untrusted-data sink in the facade.
- **`break!/4` / `__break__!` delegations** — Yes, neighbour-read. `IEx.Pry.break!/4` and `/6` confirmed present (pry.ex:237,245) matching the facade's calls.
- **AST-handling macros (`pry/0`, `break!/2`, `raise_unknown_break_ast!/1`)** — Verified by reading: all operate on developer-typed AST expanded in the shell's own env; no attacker-controlled AST path exists in this file.

## Dropped in re-review

None. All prior ruled-out entries are retained; two ("configure/1 delegation", "break!/4 defdelegate") that the first pass dismissed by "single-file review" trust reasoning are now backed by executed evidence (`color/2`) and neighbour reads (`IEx.Config`, `IEx.Pry`), raising their basis from asserted to verified without changing the conclusion. No hedged assumption rose to the level of a finding — no untrusted data enters this file's own code.
