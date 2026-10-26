# EEx (Embedded Elixir) — Correctness / Bug Profile

## Protocol / standard implemented
This is Elixir's **EEx** — Embedded Elixir templating: a tokenizer + compiler that turns `<% ... %>` template source into an Elixir quoted AST (`Macro.t()`), which is then evaluated or spliced into a compiled function. No IETF/NIST standard governs it; the "spec" is the EEx moduledoc (tag grammar) and the `EEx.Engine` behaviour contract. It is a **compiler front-end**, not a runtime protocol handler.

Public API entry points (all in `EEx`):
- `eval_string/3`, `eval_file/3` — compile then `Code.eval_quoted`.
- `compile_string/2`, `compile_file/2` — return AST only.
- `function_from_string/5`, `function_from_file/5` — macros that inject a compiled template as a module function.
- `tokenize/2` — public since 1.14; exposes the token stream.
Internals: `EEx.Compiler` (tokenizer + `generate_buffer` state machine), `EEx.Engine` (behaviour + default impl), `EEx.SmartEngine` (adds `@assign`).

## Trust / contract boundaries
- **Template source** (`source`/`filename` args) is the primary crossing. In the common Phoenix/library use, templates are **operator/developer-supplied at compile time**, NOT peer-controlled — but `eval_string`/`eval_file` explicitly support runtime evaluation, and *any* embedded expression is executed as full Elixir. Passing attacker-controlled data as the template `source` (not as a binding) is arbitrary code execution by design. The per-file reviewer should treat "is this template string ever peer-controlled?" as a caller-side question, not an EEx bug.
- **Bindings / assigns** (`bindings`, `assigns:`) are the intended data channel; values are inserted as variables, not code, so they are safe from injection but are *not* HTML-escaped by base EEx (escaping is an engine concern, e.g. Phoenix.HTML — not present here).
- **Options** are operator-controlled: `:engine` (arbitrary module invoked as a behaviour), `:trim`, `:line`, `:column`, `:indentation`, `:parser_options`. A malicious/buggy `:engine` module can return any state shape.
- **Contract callers must honour:** `compile_string` requires a binary; `tokenize` accepts charlist or binary. Engine callbacks must return the documented state map; `check_state!/1` enforces the `%{binary,dynamic,vars_count}` shape and raises otherwise.

## Failure-mode model
Because inputs are *code fragments*, the dominant failure is **raise, not silent corruption** — `EEx.SyntaxError` (with snippet) for unterminated `%>`/`--%>`, unexpected middle/end expressions, unbalanced `<% end %>`. Watch specifically for:
- **State-machine clause omissions** in `generate_buffer/4`: the scope stack (`[{contents,line,column} | scope]`) must be pushed/popped in lockstep with `start_expr`/`middle_expr`/`end_expr`. A middle/end with an empty scope list has explicit error clauses (lines 423, 441) — a *new* token kind added without a clause would `function_clause`.
- **Line/column threading errors**: line/column are recomputed by hand in `tokenize`, `expr`, `comment`, `trim_right`, `wrap_expr` (`new_lines = line - state.line`). An off-by-one propagates into error snippets and, via `wrap_expr`'s newline padding, into the *line numbers of compiled code* — silent-corruption of stack traces rather than a crash.
- **Accumulator/buffer dropped**: `handle_begin`/`handle_end` reset `binary`/`dynamic`; a nesting that fails to restore the outer buffer loses output.
- **`token_key/2` mis-classification**: the heart of block detection — decides `:start_expr` vs `:middle_expr` vs `:end_expr` vs `:expr` from re-tokenized Elixir. A legal construct classified wrong yields a spurious syntax error or a mis-nested block (see `fn_before_end?`, `drop_eol`, `closing_bracket?`).

## Common bug patterns for this family (compiler/tokenizer)
- Marker handling: `retrieve_marker` only recognises `= / |`; the `middle_expr/end_expr` + non-empty-marker path warns and strips (lines 93–103) — a refactor could drop that.
- Trim logic (`trim_left`/`trim_right`/`trim_init`) is whitespace-counting arithmetic — classic off-by-one / `\r\n` vs `\n` asymmetry territory (note `trim_left` handles `\r\n` but `trim_init`/`trim_right` branch separately).
- `merge_token` merges whitespace-only `middle_expr` runs only when `clause_block_identifier?` (`:else/:rescue/:catch`) — an omitted identifier here breaks `try/rescue` templates silently.
- `insert_quoted` walks the AST replacing `__EEX__(key)` placeholders via `Map.fetch!` — a key generated but never stored (or vice-versa) raises `KeyError`; the `{l,line,r}` vs `{l,r}` tuple arms must both recurse.
- `String.to_unsafe_atom("arg#{vars_count}")` in `EEx.Engine.handle_expr` mints atoms per expression — unbounded template size is an atom-table concern (compile-time only).

## Supervisor / process structure
**None.** EEx is pure, synchronous, stateless compile-time code — no gen_server, gen_statem, supervisor, ETS, or message passing. There is no process boundary to breach and no `handle_info` surface; every failure is an in-line `raise` in the caller's process. Threat-model accordingly: the blast radius is the compiling/evaluating process, not a supervised tree.

## Cross-file invariants
- **Tokenizer → compiler token contract:** `EEx.Compiler.tokenize` guarantees a well-formed token stream ending in exactly one `{:eof, meta}`, with `start_expr`/`end_expr` markers already normalised. `generate_buffer` relies on this — it has no defence against a malformed stream from a third-party `tokenize` caller, and `EEx.tokenize/2` now exposes that stream publicly.
- **Engine state shape:** compiler calls `init → handle_text/handle_expr/handle_begin/handle_end → handle_body` and passes the opaque state straight through. `EEx.Engine.check_state!/1` is the *only* validator; `SmartEngine` delegates everything except `handle_expr` (which prewalks `@assign`), so any engine reusing `EEx.Engine` inherits that `%{binary,dynamic,vars_count}` invariant.
- **`handle_text/2` vs `/3`:** compiler probes `function_exported?` and warns; a custom engine implementing only the deprecated `/2` still works — reviewers ruling out a text sink must check *which* arity an engine defines.
- **Assign resolution:** `@foo` is rewritten (in `SmartEngine.handle_expr` via `EEx.Engine.handle_assign`) to `fetch_assign!(assigns, :foo)`, which **warns-and-returns-nil on a missing key** rather than raising — a missing assign is silent `nil`, not a crash.