# /Users/lukas/code/claude-scanner/elixir/lib/eex/lib/eex.ex
<!-- sha256: 807aea9052f92eedc2747b586257bb93d563d719e69328050bc8ac317d4d68d5 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/eex/lib/eex.ex
<!-- sha256: 807aea9052f92eedc2747b586257bb93d563d719e69328050bc8ac317d4d68d5 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-27_

## Summary
`eex.ex` is a thin, mostly-correct public façade over `EEx.Compiler`; the only defects are a too-narrow `@spec` on `eval_file/3` and a moduledoc that describes the wrong assign-resolution semantics. Re-review read the two callgraph neighbours (`eex/engine.ex`, `eex/smart_engine.ex`) that finding #2 depends on and confirmed the warn-and-`nil` mechanism at the source level; no hedged assumptions remain.

## Bugfixes

#### `CSCAN-eex-eval-file-spec-too-narrow`
- **Finding ID** — `CSCAN-eex-eval-file-spec-too-narrow`
- **Severity** — Low
- **Confidence** — high
- **Location** — `eex.ex:318` (`@spec eval_file/3`)
- **Class** — Contract violation
- **Issue** — `eval_file/3` is spec'd `:: String.t()` (`eex.ex:318`), but its sibling `eval_string/3` is spec'd `:: term()` (`eex.ex:296`). Both funnel through `do_eval/3` (`eex.ex:367-371`), whose result is `Code.eval_quoted` of whatever the engine's `handle_body/1` returned. With the documented `:engine` option, `handle_body/1` may return any term (the `EEx.Engine` callback contract types it as `Macro.t()`, `engine.ex:33`, and there is no post-check narrowing it to a binary), so `eval_file` can return a non-binary, contradicting `String.t()`.
- **Impact** — Dialyzer is fed a false narrowing: any caller that legitimately obtains a non-string from `eval_file/3` (custom engine) gets spurious type errors, and downstream analysis that assumes the result is a binary is silently mis-informed. No runtime failure, but the contract is wrong and inconsistent with `eval_string/3`.
- **Evidence** — `verified` (executed). With a custom engine whose `handle_body/1` returns `42`:
  ```
  $ elixir eng.exs   # EEx.eval_file("/tmp/t.eex", [], engine: IntEngine)
  42
  ```
  An integer is returned where the `@spec` promises `String.t()`. The path is symmetric with `eval_string/3` (both call `do_eval/3`), which is already spec'd `term()`.
- **Suggested fix** — Change the return type to `term()` to match `eval_string/3` (`@spec eval_file(Path.t(), Code.binding(), [compile_opt]) :: term()`).

#### `CSCAN-eex-smartengine-assign-doc-mismatch`
- **Finding ID** — `CSCAN-eex-smartengine-assign-doc-mismatch`
- **Severity** — Low
- **Confidence** — high
- **Location** — `eex.ex:103-105` (moduledoc, `EEx.SmartEngine` section)
- **Class** — Documentation mismatch
- **Issue** — The moduledoc states `<%= @foo %>` "translates to" `<%= {:ok, v} = Access.fetch(assigns, :foo); v %>`. That translation *raises* `MatchError` on a missing assign (`Access.fetch/2` returns `:error`, and `{:ok, v} = :error` fails). The actual `SmartEngine` rewrite does **not** do this: `SmartEngine.handle_expr/3` prewalks the expr with `EEx.Engine.handle_assign/1` (`smart_engine.ex:58-61`), which rewrites `@foo` to `EEx.Engine.fetch_assign!(var!(assigns), :foo)` (`engine.ex:128-131`); `fetch_assign!/2` (`engine.ex:140-156`) does `Access.fetch`, and on the `:error` branch **emits `IO.warn` and returns `nil`** rather than raising. (A `# TODO: Raise on v2.0` comment at `engine.ex:138` confirms the raise is intentionally *not yet* the behaviour.)
- **Impact** — A reader relying on the documented equivalence expects a hard failure on a missing assign and will mis-reason about templates that silently render `nil`; the documented "contract" for the assign extension does not match observable behaviour.
- **Evidence** — `verified`. (a) Executed: `EEx.eval_string("<%= @foo %>", assigns: [])` prints `warning: assign @foo not available ...` and yields `""` (nil rendered), where the documented `{:ok, v} = Access.fetch(...)` form would have raised `MatchError`. (b) Source-confirmed via the callgraph neighbours read this pass: `smart_engine.ex:58-61` → `engine.ex:128-131` (`handle_assign`) → `engine.ex:140-156` (`fetch_assign!`, `:error` → `IO.warn` + `nil`). The mechanism the first pass assumed is now read, not inferred.
- **Suggested fix** — Update the illustrative translation to reflect the real semantics, e.g. describe it as `EEx.Engine.fetch_assign!(assigns, :foo)` (warn-and-`nil` on miss) rather than the raising `{:ok, v} = Access.fetch(...)` form.

### Ruled out
- **`tokenize/2` doctest map-key order** (`eex.ex:340-341`) — expected `%{column: 1, line: 1}` vs actual insertion-order inspect `%{line: 1, column: 1}`. `verified` (executed): ran `doctest EEx, only: [tokenize: 2]` → `1 passed`; doctests compare structurally, so key order is irrelevant. Axes examined: key-ordering, struct-vs-map rendering | Not examined: none.
- **Module doctests** (`eval_string/3` `eex.ex:24`, `function_from_string/5` `eex.ex:148-153`, `<%= @foo %>` `eex.ex:100`) — `verified` (executed): ran each; all produce the documented output. Axes examined: default-engine output, function-macro expansion, assign present | Not examined: none.
- **`compile_file`/`eval_file` file-read failure** (`eex.ex:282`, `eex.ex:323`) — `File.read!/1` raises `File.Error`, a normal exception with no handle to leak (it opens, reads, closes internally), matching the "propagate read failures as normal errors" convention (sub-claim NO_RESOURCE_LEAK_ON_FILE_READ). Axes examined: missing-file error surface, handle lifecycle | Not examined: none.
- **`function_from_string`/`function_from_file` double-evaluation of args** (`eex.ex:156-167`, `eex.ex:199-212`) — both use `quote bind_quoted: binding()`, so `template`/`file`/`options`/`args` are each evaluated exactly once before the quote body runs; no double side effects. Axes examined: bind_quoted single-eval semantics, arg-list re-evaluation | Not examined: none.

## Risks / caveats
- The substantive compiler/tokenizer logic (state-machine clauses, trim arithmetic, line/column threading, marker classification) lives in `EEx.Compiler`, a genuinely separate source file not under review here; `compile_string/2` (`eex.ex:238-249`) merely dispatches to `EEx.Compiler.tokenize/2` and `EEx.Compiler.compile/3` and correctly converts a `{:error, message, meta}` tuple into a raised `EEx.SyntaxError` (`eex.ex:245-247`), honouring the SYNTAX_ERROR_RAISES_EEX_SYNTAXERROR contract at this boundary. This is a file-scope boundary, not an unverified assumption about `eex.ex`'s own behaviour.
- The engine-side behaviour that finding #2 depends on is **no longer** a caveat: `EEx.Engine` and `EEx.SmartEngine` were read this pass and confirm the warn-and-`nil` assign path.
- The `:column` option is honoured by `compile_string/2` (via `Keyword.take([..., :column, ...])` at `eex.ex:239`) and documented under `tokenize/2` (`eex.ex:333`), but is absent from the module-level Options list (`eex.ex:47-70`) — a documentation gap, not a defect.
- Both findings are Low severity: neither causes a runtime crash on valid default-engine input; they are a Dialyzer-visible spec narrowing and a doc/behaviour divergence respectively.

## Did I verify this
- **Finding #1 (`eval_file` spec)** — Verified by execution (custom engine returning `42` through `eval_file/3`) plus source trace `eval_file/3 → do_eval/3` (`eex.ex:318-324`, `367-371`) and the `handle_body/1 :: Macro.t()` callback type (`engine.ex:33`). No hedge remains.
- **Finding #2 (assign doc mismatch)** — Verified by execution (missing assign warns + renders `nil`) **and** by reading the two callgraph neighbours this pass: `smart_engine.ex:58-61`, `engine.ex:128-131`, `engine.ex:140-156`. The first-pass hedge ("actual `SmartEngine` rewrite … instead emits a warning") is now backed by the read source, not assumed.
- **Ruled-out entries** — Each was cleared by execution (doctests) or by direct reading of the dispatch/quote code in `eex.ex`; every listed axis was actually tried, so all carry `Not examined: none`. Reading `engine.ex`/`smart_engine.ex` was used only to confirm a mechanism already observed at runtime — it did not, on its own, clear any axis.

### Dropped in re-review
None. Both prior findings survive with the same or strengthened evidence; no ruled-out entry was upgraded, and none was moved from `verified` to `asserted`.
