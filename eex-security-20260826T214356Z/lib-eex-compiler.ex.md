# /Users/lukas/code/claude-scanner/elixir/lib/eex/lib/eex/compiler.ex
<!-- sha256: 01b4c3e489fb47b17d7670bbc0178e3169f6d14511a2e9cf912a62c88d67c37e -->

_Analysed on 2026-08-26T21:57:30Z_

I have everything I need. The analysis is complete and the primary finding is empirically verified.

## Summary

`EEx.Compiler` is safe on its intended (trusted-template) compile/eval paths, but its per-expression tokenizer classification interns a permanent atom for every identifier inside `<%…%>` tags, turning the non-executing, documented `EEx.tokenize/2` API into an atom-table–exhaustion DoS on untrusted template input.

## Security

#### `CSCAN-compiler-tokenize-atom-exhaustion`

- **Finding ID** — `CSCAN-compiler-tokenize-atom-exhaustion`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `compiler.ex:83`, in `defp tokenize(~c"<%" ++ t, line, column, state, buffer, acc)` — the `:elixir_tokenizer.tokenize(expr, 1, file: "eex", check_terminators: false)` classification call.
- **Class** — Atom creation (CWE-770, unbounded resource consumption)

- **Issue** — For every expression tag (`<%`, `<%=`, `<%|`, `<%/`), the tokenizer runs the full Elixir tokenizer over the tag body purely to classify it as start/middle/end/plain (`token_key/2`). The classification only inspects fixed keyword token *types* (`:do`, `:end`, `:block_identifier`, `:stab_op`, `:fn`, closing brackets), but the tokenizer is invoked with default options, so it **interns a new atom for every identifier, alias, and `:atom` literal** in the tag body as a side effect. This runs inside `EEx.tokenize/2` (eex.ex:361, `@doc since: "1.14.0"`, public) — the one documented entry point that does **not** compile or evaluate the template. `<%%` quotations (compiler.ex:39) and `<%!-- --%>` comments (compiler.ex:43) are handled before line 72 and do not hit this sink.

- **Impact** — An application that treats `EEx.tokenize/2` as a side-effect-free lexer — a formatter, linter, syntax highlighter, LSP, or template-preview tool inspecting untrusted `.eex` content *without running it* — leaks one permanent, never-GC'd atom per unique identifier in the input. Atoms are a global VM resource with a hard default cap (~1,048,576); either one large hostile template or an accumulation of many small ones (atoms persist across calls) exhausts the table and aborts the entire BEAM node with `system_limit`. The victim's assumption "lexing doesn't execute anything, so it's safe on untrusted input" is silently false. (On the compile/eval paths this is dominated by the fact that a template is arbitrary code = RCE by design, so `tokenize` is the distinct, non-RCE surface where this matters.)

- **Evidence** — `verified`.
- Interning + growth via `EEx.tokenize` (no compile/eval): template of 20,000 unique identifiers → `atoms before: 18632 / atoms after: 38671 / delta: 20039`, and `:zzuniqident1` became a live atom (`list_to_existing_atom` succeeded).
- Mechanism isolated (10,000-tag controls): `text-only: delta=40`, `no-marker <% id %>: delta=10000`, `=-marker <%= id %>: delta=10000`, `atom literals :atom: delta=10000`, `<%% quotation %>: delta=0`, `<%!-- comment --%>: delta=0` — growth occurs exactly for the tags that reach line 83.
- Classification needs only keyword atoms: tokenizing `" if x do "` yields `{:identifier,…,:if}, {:do_identifier,…,:x}, {:do,…}` — `token_key` branches on the token *type*, never on the interned identifier value.

- **Suggested fix** — Stop interning identifiers during the classification pass: pass a `:static_atoms_encoder` to the `:elixir_tokenizer.tokenize` call at line 83 that resolves already-existing atoms (`String.to_existing_atom/1`) and returns a non-interned placeholder term for everything else. This preserves classification, because the keyword atoms `token_key/2` relies on (`:do`, `:else`/`:rescue`/`:catch` block identifiers, `:stab_op`, `:fn`) always already exist while novel template identifiers are the ones that must not be interned. Note the verified subtlety: `existing_atoms_only: true` alone is **not** a correct fix — it makes the tokenizer return `{:error, …}` on a novel identifier (I observed `unsafe atom does not exist: brandnewvar_1442`), which drops to the `{:expr, …}` branch at line 90 and misclassifies legitimate blocks such as `<% if newvar do %>`.

## Risks / caveats

- **Scope of the finding.** The compile/eval/`function_from_*` entry points also intern atoms (both at line 83 and via `Code.string_to_quoted!` at lines 372/436), but those paths execute the template as code — arbitrary RCE by EEx's design, per the folder threat model — so atom exhaustion adds no marginal risk there. The finding is specifically about `EEx.tokenize/2`, the documented API that lexes without executing. Its severity is Medium rather than High because it depends on the (plausible but not universal) pattern of tokenizing untrusted templates; the mechanism itself is certain.
- **SSTI/RCE intentionally not reported.** Feeding an attacker-controlled *template* to `compile_string`/`eval_string`/`function_from_string` is arbitrary code execution, but that is EEx's documented contract (the template is code, there is no sandbox), and it requires the caller to route untrusted data into the template-source channel — the caller's responsibility, not a defect in this file.
- **Missing HTML/output escaping is out of scope** by design: `SmartEngine` inserts only a `String.Chars` conversion; XSS defense belongs to downstream engines (`Phoenix.HTML`), not this file.
- I did not exhaustively fuzz the tokenizer/`generate_buffer` for algorithmic-complexity blowups (e.g. `++`-chained `merge_token`, nested-block `:lists.flatten`); those live on the compile path (already RCE) and a quick structural review showed no clearly super-linear behavior reachable from `tokenize`, but this was reasoned, not fuzzed.

### Ruled out
- **`code_snippet` `String.duplicate(" ", meta.column - 1)` underflow (line 570) / `String.duplicate(" ", indentation)` (line 563).** Verified that `EEx.compile_string("<% foo", column: 0)` raises `ArgumentError: 2nd argument out of range` instead of a clean `EEx.SyntaxError`, and `indentation: -5` does the same. But this is driven by the caller-supplied `:column`/`:indentation` options, not by the untrusted template channel — template content alone (default `column: 1`) always produced a clean `EEx.SyntaxError` in testing. `:column` accepts `0` under `@type column :: non_neg_integer`, but the entity choosing that value is the trusted caller, and the only consequence is a different exception class on an already-failing error path. Not the attacker's channel; no memory/logic-safety impact.
- **`insert_quoted` `Map.fetch!(quoted, key)` KeyError via `__EEX__(N)` in a tag (lines 470/510-512).** A template author could write `__EEX__(999)` inside a block expression and cause a `KeyError` when the placeholder round-trips, but this is compile-path only (never reached by `tokenize`) and requires control of the template *as code*, where the author can already run anything. No security uplift.
- **Engine dispatch (`state.engine.init/handle_text/handle_expr/handle_begin/handle_end/handle_body`, lines 334/358/361/447/472/545) and `:parser_options` passthrough (lines 321/330).** The engine module and parser options come from `opts[:engine]`/`opts[:parser_options]` — operator-controlled configuration, not attacker-forwarded data.
- **`Code.string_to_quoted!` (lines 372, 436).** Parses the template's embedded Elixir; the template is trusted-as-code on every path that reaches these (compile/eval), so this is RCE-by-design, not a defect.
- **`<%%` quotation / marker handling (lines 39-41, 153-159).** Produces literal `<%…%>` text as documented; no downstream interpreter or trust decision hangs off it.
- **Unterminated `comment/5` and `expr/5` loops (lines 163-195).** Both recurse on a strictly shrinking list with a `[]` base case returning `{:error, …}`; no unbounded loop.

