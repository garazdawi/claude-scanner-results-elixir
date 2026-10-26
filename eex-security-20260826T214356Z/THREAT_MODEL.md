I have enough to write the threat model. EEx is a well-understood template-injection surface; no web search needed.

```markdown
# Threat Model: EEx (Embedded Elixir) — `elixir/lib/eex`

## Protocol / standard implemented
This is **EEx**, Elixir's template compiler. It embeds Elixir code inside text via
tags (`<% %>`, `<%= %>`, `<%% %>`, `<%!-- --%>`, plus engine-defined `<%| %>` / `<%/ %>`)
and **compiles the template into an Elixir AST that is then executed**. There is no
network protocol or RFC governing this; the governing "spec" is the EEx tag grammar
(this module's own docs) and the Elixir tokenizer/parser it delegates to
(`:elixir_tokenizer.tokenize/4`, `Code.string_to_quoted!/2`).

Public API entry points (`EEx`):
- `eval_string/3`, `eval_file/3` — compile **and run** a template at runtime.
- `function_from_string/5`, `function_from_file/5` — compile a template into a module function at build time (macros).
- `compile_string/2`, `compile_file/2` — return quoted AST (caller runs it).
- `tokenize/2` — expose the token stream.
- Engine behaviour (`EEx.Engine`) + default `EEx.SmartEngine`, with `handle_assign/1`/`fetch_assign!/2` for `@assigns`.

## Trust / contract boundaries
The single most important fact: **the template source is CODE, not data.** Everything
inside `<%…%>` is parsed by the Elixir parser and executed with the full privileges of
the calling process. The critical boundary question at every call site is *who controls
the template string*:
- **Template source (`source`/`filename`)** — if this crosses a trust boundary (user-submitted template, uploaded `.eex` file, DB-stored template), it is **arbitrary remote code execution**, not "template output escaping." EEx does *no* sandboxing; `compile_string` will happily compile `<%= System.cmd("rm",…) %>`.
- **`bindings` (eval_*) / function args** — data supplied to an *already-compiled* template. Peer-controlled data is normally expected here; this is the safe channel.
- **`options`** — operator-controlled: `:file`, `:line`, `:column`, `:indentation`, `:trim`, `:engine`, `:parser_options`. `:engine` selects arbitrary code (the engine module runs during compile). `:parser_options` is forwarded verbatim to `Code.string_to_quoted!` (e.g. `:static_atoms_encoder`, `:existing_atoms_only`).
- **`@assigns`** — `fetch_assign!` uses `Access.fetch`, so `assigns` must implement `Access`; a missing key only warns and returns `nil` (does **not** raise — TODO notes v2.0 will).

Implicit contract: callers of `eval_string`/`compile_string` **must** treat the template
as trusted. Nothing in the code enforces this — the contract lives only in intent.

## Attacker model
- **In scope:** a caller who mistakenly feeds attacker-controlled text as the *template* → RCE / SSTI. Also: crafted template triggering a **compiler crash / uncaught exception / unbounded resource use** at compile time (parser DoS, deep nesting, atom exhaustion via generated atoms).
- **Out of scope (per EEx's design):** EEx is **not** an output-encoding/HTML-escaping engine — the default `handle_expr("=")` calls `String.Chars.to_string/1` with **no HTML escaping**. XSS defense is the *downstream* engine's job (e.g. Phoenix.HTML's engine), not this folder. Do not flag "missing escaping" as an EEx bug; it is by design.
- Not a network-facing component; no post/pre-auth distinction. The BEAM co-tenant threat is the same as any compile-time macro.

## Common attack patterns for this family
- **SSTI → RCE:** untrusted template = code execution. The canonical class for template engines (Jinja2/ERB/Twig SSTI equivalents).
- **Atom exhaustion:** `String.to_unsafe_atom("arg#{vars_count}")` in `EEx.Engine.handle_expr` and any `String.to_atom` in generated code — non-GC'd atom table growth. Bounded here by expression count in one template, but relevant if templates are large/adversarial.
- **Parser/tokenizer DoS:** deeply nested `<%= if … do %>` blocks, unterminated tags (`expr/comment` loops to EOF), pathological `:parser_options`.
- **`<%%` quoting bypass / marker confusion:** `retrieve_marker` + `<%%` literal handling; mis-tokenization could change what is code vs text.
- **`code_snippet` index math:** error-path string slicing on `meta.column`/`line` (`column - 1` for arrow padding) on adversarial column values.

## Supervisor / process structure
**None.** This folder contains no gen_server, gen_statem, or supervisor. All modules are
pure compile-time/functional transforms invoked synchronously in the caller's process.
There is therefore no process boundary acting as a crash-containment barrier — an
uncaught error in `EEx.Compiler` propagates directly to whichever process called
`eval_*`/`compile_*`/`function_from_*` (at build time, that is the *compiler* process).

## Cross-file invariants (what a per-file reviewer would miss)
1. **Tokenizer → Compiler handoff:** `EEx.Compiler.tokenize/*` produces the `EEx.token()`
   shape; `generate_buffer/4` in the same file trusts that shape (markers ∈ `[]|=|/||`,
   balanced `start/middle/end_expr`). A reviewer ruling on `compile/3` must know structural
   validity comes only from `tokenize`, not from a schema.
2. **Two-stage parsing:** `tokenize` runs `:elixir_tokenizer` with `check_terminators: false`
   purely to *classify* an expr as start/middle/end (`token_key`); the **authoritative**
   parse is `Code.string_to_quoted!` in `generate_buffer`. Bugs in `token_key` cause
   mis-nesting, not the final AST semantics.
3. **Engine trust:** `Compiler` calls `state.engine.{init,handle_text,handle_expr,handle_begin,handle_end,handle_body}` and trusts each to return a valid buffer state; `EEx.Engine.check_state!/1` is the *only* guard, and it only checks map shape. `SmartEngine` delegates everything except `handle_expr`, which prewalks `handle_assign` — so `@x` → `fetch_assign!(assigns, :x)` requires an `assigns` var in scope at the eval site (invariant assumed by `Engine`, satisfied by the caller's bindings, not by this folder).
4. **`__EEX__` placeholder round-trip:** `wrap_expr` emits `__EEX__(key)` text that is
   re-parsed by `Code.string_to_quoted!` and then `insert_quoted` swaps it back for the
   real nested AST. The invariant is that `__EEX__` never collides with user code — a
   reviewer seeing `insert_quoted`'s `Map.fetch!` must know keys come only from `wrap_expr`.
5. **No sanitization anywhere:** no file in this folder validates that template *content*
   is safe; every entry point assumes the caller already established template trust.
```