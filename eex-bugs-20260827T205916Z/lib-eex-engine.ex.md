# /Users/lukas/code/claude-scanner/elixir/lib/eex/lib/eex/engine.ex
<!-- sha256: 863d0beb23bb19e97c3b2e0cec8cf374223ba2a146aa139328e23319b2c4c66e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:06:25Z_

## Summary
`EEx.Engine` is a small, well-behaved default-engine implementation; I found no bug that produces a crash, wrong value, hang, leak, or contract violation through its documented public API.

## Bugfixes

No findings.

### Ruled out

- **`handle_expr` marker `"/"`/`"|"` raise (engine.ex:218–221)** — I suspected `raise EEx.SyntaxError, "<string>"` (no `:file`/`:line`/`:column`/`:snippet`) might blow up or produce a message-less error, violating `SYNTAX_ERROR_RAISES_EEX_SYNTAXERROR`. *Verified* clean: `elixir -e 'EEx.compile_string("<%/ foo %>")'` raises `** (EEx.SyntaxError) nofile:1:1: unsupported EEx syntax <%/ %> ...` with a full snippet — the compiler rescues and enriches position info, and `EEx.SyntaxError.message/1` guards `snippet || ""`. Contract honored.
- **Missing-assign path in `fetch_assign!/2` (engine.ex:140–156)** — checked whether a missing assign crashes instead of the documented warn-and-`nil`. *Verified*: `EEx.eval_string("<%= @bar %>", assigns: [foo: 5])` emits the `assign @bar not available` warning and renders `""` (from `String.Chars.to_string(nil)`); `SMART_ENGINE_ASSIGNS_CONTRACT` satisfied.
- **State threading / `handle_body` ordering (engine.ex:179–216)** — verified `EEx.eval_string("a <%= x %> b", x: 1)` returns `"a 1 b"`; the binary `<<>>` node is placed last in the block and text/expr accumulators are reversed correctly, so `ENGINE_STATE_THREADED` / `HANDLE_BODY_RETURNS_QUOTED` hold.
- **Deprecated `__using__` macro declares `@behaviour EEx.Engine` but defines only `handle_text/2`, not `/3` (engine.ex:80–110)** — this is a genuine wart: every `use EEx.Engine` module gets a spurious compile-time `function handle_text/3 required by behaviour EEx.Engine is not implemented` warning (*verified* via a `MyEngine` test module). But the whole macro is `@deprecated`, the compiler's `function_exported?` probe falls back to the `/2` clause, and output is correct (`"hello 3"`). No crash, wrong value, or output-affecting contract breach — cosmetic only, on a deprecated path, so not reported as a finding.
- **`handle_assign` guard `is_atom(name) and is_atom(atom)` (engine.ex:128)** — considered mis-rewriting non-assign `@`-forms; the guard correctly matches only bare `@foo` reads (`{:foo, _, nil}`) and passes attribute-writes (`{:foo, _, [arg]}`) through unchanged. No defect.

## Risks / caveats
- `String.to_unsafe_atom("arg#{vars_count}")` (engine.ex:198) mints one atom per `<%= %>` expression; this is a compile-time-only, template-size-bounded concern already noted in the folder threat model, not reachable as unbounded growth via a documented runtime entry point, so it is not a finding here.
- `check_state!/1` is invoked in `handle_begin`/`handle_body`/`handle_text`/`handle_expr "="` but not in the `handle_expr ""` clause (engine.ex:213–216); this is a harmless asymmetry — a malformed state only reaches these functions by an engine violating the documented `%{binary,dynamic,vars_count}` contract, i.e. internal misuse out of scope.
- Analysis was confined to `engine.ex`; the tokenizer/compiler boundary that feeds these callbacks lives in `compiler.ex` and was not reviewed here.

