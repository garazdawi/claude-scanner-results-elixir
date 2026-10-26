# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/formatter.ex
<!-- sha256: a06ef151762a210f272d014db856f9756c4261b0c876440ccd9781b4a87a0ade -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/formatter.ex
<!-- sha256: a06ef151762a210f272d014db856f9756c4261b0c876440ccd9781b4a87a0ade -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`ExUnit.Formatter` is a byte-for-byte match of the compiler's own built stdlib copy. All five doctests reproduce exactly when run, and close reading of every public entry point turned up no defect reachable through a documented, contract-conforming call. The two paths the first pass left un-driven — the `{:mailbox, ...}` branch of `format_assertion_diff/4` and the three `linked_or_trapped_exit` branches of `format_kind_reason/6` — have now been executed end-to-end against the built ExUnit (transcripts below); both format cleanly and neither raises. No findings.

## Bugfixes

No findings.

## Risks / caveats

- The file under review is **identical** to the compiler's built stdlib copy (`diff` empty), so I could not use a pristine-upstream baseline to isolate an injected change — every conclusion rests on direct behavioral testing and reading, not on a reference diff. This is a methodology limitation, not a hedge about a specific code path.

- **Upgraded from hedged assumption in first pass** (now resolved, no finding): the first pass wrote that it "did not exhaustively exercise `format_assertion_diff/4`'s `{:mailbox, ...}` branch or the linked/trapped-exit paths of `format_kind_reason`." I drove all of these with live inputs:
  - `format_assertion_diff/4` with `context: {:mailbox, [], [{:msg, 1}, {:other, 2}]}` → returns `[mailbox: <iodata>]` that renders as paired `pattern:`/`value:` lines. Empty-mailbox `context: {:mailbox, [], []}` → returns `[]` (the guard clause at formatter.ex:446). Both correct.
  - `linked_or_trapped_exit({:EXIT, pid}, {%RuntimeError{}, stack})` (formatter.ex:358) → `** (EXIT from #PID<...>) an exception was raised:` header with the nested banner and padded stacktrace. Correct.
  - `linked_or_trapped_exit(:exit, {{%RuntimeError{}, stack}, {mod, fun, args}})` (formatter.ex:364) → `** (exit) exited in: MyMod.myfun(1, 2)` followed by the nested exception. Correct.
  - The `reason == :function_clause` OR-guard alternative (formatter.ex:360, :367) via `{{:EXIT, pid}, {:function_clause, stack}}` → matches, normalizes to `FunctionClauseError`, renders without raising. This also confirms the `reason.__struct__ in @formatter_exceptions` sub-guard fails safe (returns false, not a crash) when `reason` is a bare atom, because the OR alternative carries it.

## Ruled out

- **`with_location/1` crashes when `tags[:file]` is nil** (formatter.ex:712–713) — `Path.relative_to_cwd(nil)` raises `FunctionClauseError` (verified by execution). Reachable via `format_test_failure/5` only if the caller passes an `ExUnit.Test` whose `tags` map omits `:file`/`:line`. Real ExUnit tests always populate these tags and the doctest supplies them; a struct without them violates the implicit `tags` contract — misuse of the interface, not a public-API bug. Confidence that this is not a reachable defect: high. Axes examined: nil-file input value. | Not examined: none (the nil value is the only axis; pushing it through raises, and the reachability question is a contract question, already settled).

- **`get_code/2` arithmetic on a nil line** (formatter.ex:577–592) — when a stack entry lacks `:line`, `line > 0` evaluates `nil > 0` → `true` (atom/nil > integer, verified), and `Enum.at(stream, nil - 1)` raises `ArithmeticError` (verified). The entire body is wrapped in `rescue _ -> nil` (formatter.ex:590–591), so no error escapes and the function correctly yields `nil`. Not a defect. Confidence: high. Axes examined: nil-line value through the guard and the arithmetic, plus the rescue. | Not examined: none.

- **`format_us/1` truncation / difference-of-truncations in `sync_us`** (formatter.ex:203, :220–227) — normalized centisecond truncation means `run_us - async_us` can differ from a truncation of the true difference by one unit. This is the intended, self-consistent design; all four `format_times/1` doctests including the async case (formatter.ex:194–195) pass exactly. Not a miscomputation. Confidence: high. Axes examined: the four documented run/async/load combinations. | Not examined: none material — the doctest set spans nil-async, nil-load, both-present, and sub-10cs/over-10cs formatting.

- **`Enum.map_join(stacktrace, fn …)` (2-arg form)** (formatter.ex:699) — resolves to `map_join(enumerable, mapper)` with default joiner `""`, the intended behavior; not a missing-argument bug. Confidence: high. Axes examined: arity resolution. | Not examined: none.

- **`&code_multiline(&1, padding_size)` inside the `code_multiline = if …` binding** (formatter.ex:381–384) — this is function-capture syntax referring to the private `code_multiline/2` (formatter.ex:669–675), not the yet-unbound local variable; no self-reference / infinite recursion. Verified live: the `{:EXIT, pid}` transcript above exercises `format_exception` → this binding without looping. Confidence: high. Axes examined: capture-vs-local resolution, executed. | Not examined: none.

## Did I verify this?

- Read the full file (785 lines) and the relevant `@doc`/`@spec` contracts for every public entry point.
- Executed the `{:mailbox, ...}` non-empty and empty branches of `format_assertion_diff/4` — both format correctly (empty → `[]`).
- Executed all three `linked_or_trapped_exit` branches (`{:EXIT, pid}` with exception struct, `:exit`-in-mfa, and the `:function_clause` atom OR-guard) via `format_test_failure/5` — all render without raising.
- Re-confirmed the `Path.relative_to_cwd(nil)` and `get_code/2` nil-line raises by execution, and confirmed the `rescue` on `get_code/2` swallows the arithmetic error.
- Toolchain used: Elixir 1.21.0-dev on Erlang/OTP 29, running against the built stdlib ExUnit (which is this exact source).

## Dropped in re-review

No findings were dropped. The prior report contained no findings; its single hedged caveat ("did not exhaustively exercise …") has been resolved to a verified, non-finding statement by driving the paths through the real code.
