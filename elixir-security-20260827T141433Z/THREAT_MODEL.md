```markdown
# Threat model: `elixir/lib/elixir/src` (the Elixir compiler front-end)

## Protocol / standard implemented
This is the **Erlang-implemented core of the Elixir compiler**: the pipeline that turns Elixir source text into Erlang abstract format for the BEAM. No network/wire RFCs govern it. The relevant "standards" are internal contracts: the **Elixir language grammar** (`elixir_parser.yrl`, a yecc grammar), **Unicode** (UAX #31 identifier/annex handling, bidi/`?bidi` and line-break checks in `elixir_interpolation.erl`, UTF-8 via `unicode_util`), and Erlang abstract format / `erl_eval` semantics. The `URIConfig` block in `elixir.erl` seeds default ports (RFC 3986 schemes) into config but does no parsing here.

Pipeline: `elixir_tokenizer` → `elixir_parser` (yrl) → `elixir_expand` (macro expansion) → `elixir_erl*` (translation to Erlang forms) → `elixir_erl_compiler`/`elixir_compiler`. Support modules: `elixir_aliases`, `elixir_import`, `elixir_dispatch`, `elixir_module`, `elixir_def`, `elixir_quote`, `elixir_map`, `elixir_bitstring`, `elixir_clauses`, `elixir_rewrite`, `elixir_errors`.

**Public API entry points** (all documented as *private to the compiler*, but callable): `elixir:string_to_tokens/5`, `tokens_to_quoted/3`, `string_to_quoted/5`, `'string_to_quoted!'/5`, `eval_quoted/3,4`, `eval_forms/3,4`, `env_for_eval/1`, `quoted_to_erl/2`. `elixir:start/2` is the OTP application callback; `start_cli/0`/`start/0` boot the CLI/IEx.

## Trust / contract boundaries
- **Source text is the primary untrusted input.** `String` reaching `string_to_tokens`/`string_to_quoted` is attacker-controlled when a tool compiles or parses code from an untrusted source (formatters, language servers, `Code.string_to_quoted/2` on user data, doc/markdown tooling). The tokenizer must treat every byte as hostile: unterminated strings, malformed UTF-8, bidi/line-break injection, huge/deeply-nested input, adversarial `\u{...}`/`\xHH` escapes (`elixir_interpolation.erl`).
- **`eval_*`/`quoted_to_erl` execute code.** Callers are contractually responsible for only evaluating trusted quoted forms — these functions *run* arbitrary code (`erl_eval:expr`). They are not a sandbox; treating a quoted AST from an untrusted source as safe to `eval_forms` is a caller bug, not defended here.
- **`env_for_eval/1` contract**: expects a `Macro.Env` struct with `map_size == 15` or an opts list; a caller passing a hand-built env violates the implicit shape contract.
- **Operator/config-controlled**: `elixir_config` table entries (compiler options: `tracers`, `parser_options`, `on_undefined_variable`, `erlc_options`). `Opts` proplists (`literal_encoder`, `columns`, `token_metadata`) are caller-set, stored in the **process dictionary** (`put(elixir_parser_warnings...)` etc.) — parsing is single-process, non-reentrant across those keys.

## Attacker model
The realistic attacker is **a supplier of untrusted Elixir source or an untrusted quoted AST** to a host process that parses/compiles it. In scope: DoS of the compiling process (crash, unbounded CPU/memory, atom-table exhaustion), and confusing diagnostics. Explicitly **out of scope**: sandboxing evaluation — `eval_*` is documented to run code; and defending against a malicious *operator* who already controls compiler config/tracers (they already have code execution). A co-tenant BEAM process is not a boundary the compiler defends. Note the header on `elixir.erl`: these are "private to the Elixir compiler."

## Common attack patterns for this code family
Recognise these when reviewing a specific path:
- **Atom exhaustion** — `list_to_atom`/`binary_to_atom` on peer-derived text (compiler module names in `elixir_code_server:compiler_module/1` are bounded; watch tokenizer/aliases paths that atomise identifiers).
- **Unbounded recursion / stack blowup** — deeply nested delimiters, interpolation nesting (`extract/8` recurses into `elixir_tokenizer:tokenize` per `#{...}`), quoted-tree walks in `elixir_quote`/`elixir_expand`.
- **Quadratic blowup** — repeated `<<Acc/binary, ...>>` or list-reverse patterns on large input (unescape loops, tokenizer buffers).
- **Bidi / homoglyph / line-break injection** (CVE-2021-42574 "Trojan Source") — explicitly guarded in `extract_char` via `?bidi`/`?break`; verify no path bypasses it (raw `unescape_*`, sigils with `Interpol=false`).
- **Escape-sequence edge cases** — `\u{}`/`\x` truncation, reserved/invalid codepoints (`append_codepoint` catches `badarg`), integer overflow via `list_to_integer/2`.
- **Macro-expansion / hygiene escape** — `elixir_quote` var counters and `elixir_expand` context; unhygienic capture is a correctness/security concern for generated code.
- **Process-dictionary leakage across nested eval** — `elixir.erl` carefully saves/restores `?elixir_eval_env` and `dbg_callback`; a missing restore leaks env between evals.

## Supervisor / process structure
`elixir_sup` (`one_for_one`, 3 restarts / 10s) supervises exactly two **permanent workers**, both singletons:
- **`elixir_config`** — holds the compiler config ETS/table (options above). Crash loses global compiler config.
- **`elixir_code_server`** (`gen_server`) — serialises module definition (`defmodule`/`undefmodule`, monitors compiling PIDs via `'DOWN'`), the `required`-files acquire/release queue, and the compiler-module pool. A crash here aborts all in-flight compilation and drops queued `acquire` waiters. Note `handle_call`/`handle_cast` deliberately `{stop, {badcall/badcast, ...}}` on unknown messages — an unexpected message **takes the server down** (and with it, siblings' compilation state). Actual tokenizing/parsing/expansion runs in the **caller's process**, not under this supervisor — a parse crash kills only the caller.

## Cross-file invariants
- **`elixir_tokenizer` is the sole sanitiser of raw bytes.** `elixir_parser`, `elixir_expand`, and `elixir_erl*` assume tokens are well-formed (valid UTF-8, bidi/break already rejected). A sink in the parser that looks unguarded is usually guarded because the tokenizer refused the input first — but only for the *default* tokenize path; `elixir_interpolation:unescape_string/1` and sigils with `Interpol=false` re-enter unescaping **without** the `?bidi/?break` gate.
- **`elixir_interpolation:extract/8` re-enters `elixir_tokenizer:tokenize/4`** for each `#{...}`; nesting depth = source-controlled recursion depth across both files.
- **Parser state lives in the process dictionary** (`elixir_parser_warnings/columns`, `elixir_token_metadata`, `elixir_literal_encoder`), set by `put_parsing_state` and erased in `tokens_to_quoted`'s `after`. Any file reading these keys assumes a `tokens_to_quoted` call set them; concurrent parsing in one process would corrupt them.
- **`env_for_eval` normalises the env** (validates PID liveness, strips tracers for dead lexical_tracker) before `eval_forms`/`quoted_to_erl` translate — downstream translation assumes a live-or-nil `lexical_tracker`.
- **`elixir_code_server` is the single writer of the `elixir_modules` ETS table**; `defmodule` rejects duplicates (`[CurrentTuple] -> {error, ...}`), so other files may assume module-name uniqueness is enforced centrally, not per-caller.
```