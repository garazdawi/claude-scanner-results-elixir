This is the Elixir compiler front-end (tokenizer → macro expander → Erlang abstract-code generator), not a network protocol. I have enough to write the summary. Note the Trojan Source / bidi handling I saw in the interpolation module.

# Folder correctness/bug profile: `elixir/lib/elixir/src` (Elixir compiler front-end, written in Erlang)

## Protocol / standard implemented
This is **not** a network protocol implementation — it is the **Elixir language compiler front-end**, written in Erlang. It turns Elixir source text into Erlang abstract format for the BEAM. The relevant "specs" are the **Elixir language semantics** (macros, hygiene, sigils, structs) and the **Erlang Abstract Format** it emits (`erl_parse` trees consumed by `compile`/`erl_eval`). Unicode handling follows the **Unicode Standard** (UAX #31 identifiers, UAX #15 normalization, UAX #9 bidi — the `?bidi`/`?break` guards in `elixir_interpolation` are the **Trojan Source / CVE-2021-42574** mitigation). Pipeline stages: `elixir_tokenizer` → yecc parser (`elixir_parser`, generated, not in folder) → `elixir_expand`/`elixir_clauses`/`elixir_quote` (macro expansion & hygiene) → `elixir_erl*` (Erlang code gen).

Public API entry points (all documented "private to the compiler" but *are* the callable surface): `elixir:string_to_tokens/5`, `tokens_to_quoted/3`, `string_to_quoted/5`, `'string_to_quoted!'/5`, `eval_forms/3,4`, `eval_quoted/3,4`; `elixir_tokenizer:tokenize/1,3,4`; `elixir_interpolation:unescape_string/1,2`, `unescape_tokens/1`; `elixir_config:get/1,2`, `put/2`; `elixir_code_server:call/1`, `cast/1`.

## Trust / contract boundaries
- **Source text is fully attacker-controlled** at `tokenize`, `string_to_quoted`, `unescape_string`. The tokenizer/interpolation code is the *primary* fuzz/hostile-input surface: arbitrary bytes, arbitrary length, arbitrary Unicode, malformed escapes, unbalanced terminators. Contract is essentially "any binary/charlist" — so a `function_clause` or `badmatch` here is a real crash-on-stray-input bug, not a contract violation.
- **`Line`/`Column`/`Scope` accumulators** thread through every tokenizer/interpolation clause; they are library-internal but derived from input length. Off-by-one column arithmetic (e.g. `Column+2`, `Column+3`, `Column+4` literals in `elixir_interpolation`) is silent-corruption territory — wrong diagnostics locations, not crashes.
- **`elixir_config`/`elixir_code_server`** take operator/build-tool config and compiler-internal state. `elixir_config:get/1` (line 24) deliberately has **no fallback clause** — a lookup miss `badmatch`es by design; callers must guarantee the key exists.
- **`unescape_*` contract**: throws `{error, Reason, Token}` internally, caught and reraised as `ArgumentError`; a new throw shape that escapes the `try` in `unescape_tokens/1` (line 144) or `unescape_string/2` (line 163) would leak.

## Failure-mode model
- **Crash-on-stray-input** in tokenizer/interpolation: a legal-but-rare byte sequence hitting a clause gap. The interpolation `extract/8` catch-all (lines 90–91) and `extract_char` unicode path (lines 93–112) are the safety net; a missing clause upstream falls through to them, but a *wrong* early clause silently mis-tokenizes.
- **Accumulator dropped on edge-case clause**: `finish_extraction` (line 285) special-cases empty buffer to `[[]]`; `build_string`/`build_interpol` reversal order matters — a dropped `Output` on the heredoc/newline path (`extract_nl`, lines 116–124) yields silently truncated strings.
- **State-threading (`St0` vs `St1`)**: pervasive in `elixir_expand`, `elixir_clauses`, `elixir_module` where an env/scope record is passed and returned through every expansion clause; passing the pre-expansion env forward loses variable/hygiene counters silently.
- **Missing `handle_info`/unhandled message**: both gen_servers here have catch-alls (`elixir_code_server` line 137, `elixir_config` `handle_cast` line 91 *stops* on unknown cast) — note the asymmetry: config crashes on bad cast, code_server ignores bad info.
- **Contract-shape violations** at `elixir_erl*`: returning an abstract-format tuple the BEAM `compile` step rejects — surfaces as an opaque later error, not at the point of the bug.

## Common bug patterns for this family (compiler front-end)
- **Column/line off-by-ones** from hard-coded `+N` after multi-char tokens (heredocs, sigils, `#{` interpolation, escaped delimiters).
- **Guard-macro precedence typos**: the `?..._op` macros in `elixir_tokenizer` mix `;` (clause-alt) and `orelse`/`andalso` (e.g. `?mult_op` line 36 uses `orelse`, `?rel_op` line 55 uses `;`) — a wrong connective silently changes which chars tokenize as an operator.
- **Bidi/breaking-char bypass**: any string path reaching `unescape_chars` *without* the `?bidi`/`?break` guard reintroduces Trojan-Source.
- **Hygiene/counter leaks** in `elixir_quote`/`elixir_expand`: reusing a var counter across expansions → variable capture.
- **Heredoc/sigil terminator mismatch**: `extract` matches `Last` (single char) vs `[H,H,H]` (triple) — a clause ordering change breaks nested/triple-quoted strings.

## Supervisor / process structure
`elixir_sup` (`one_for_one`, 3 restarts/10s) supervises exactly two permanent workers: **`elixir_config`** (gen_server guarding the `elixir` ETS config table + persistent_term) and **`elixir_code_server`** (gen_server guarding the `elixir_modules` ETS table, module-def registration via process monitors, and the compiler-module pool). Both own **named public ETS tables**; if either crashes, its table is lost and every concurrent compile process depending on it fails. `elixir_code_server` also owns `'DOWN'`/`{purged,...}` monitor messages — an unhandled monitor reply shape would leak module refs (memory growth), not crash. The rest of the folder is **stateless library code** run inside caller (compiler) processes — no supervision, so a tokenizer/expander crash takes down only the one file's compile job.

## Cross-file invariants
- **`elixir_interpolation:extract` reenters `elixir_tokenizer:tokenize`** for `#{...}` (line 63) with `terminators=[]`; it relies on the tokenizer returning the specific `{error, {_, _, "}"}, [$}|_], ...}` shape to resume. A tokenizer change to that error tuple silently breaks interpolation resumption.
- **`elixir_config:get/1` callers must pre-insert keys**; only `get/2` is crash-safe. Ruling out a crash in a consumer requires confirming the key was `put` at startup (`elixir:start/2`, lines 82–105).
- **Config is a singleton per node** (`?MODULE`-named table & registered gen_server); code assuming re-entrancy or per-compile isolation is wrong — writes serialize through the one gen_server (`serial/1`, `put/2`).
- **`unescape_map/1` is the single source of truth** for which escapes are legal; `unescape_chars` trusts it totally (`Map(Escaped)` line 209). A caller passing a custom map (sigils do) changes escape semantics without the tokenizer knowing.
- **`elixir_code_server` module registration** assumes each module is registered once (`ets:lookup` empty-check, line 35) and un-registered via monitor `'DOWN'`; a compile process that dies without the monitor firing leaves a stale `elixir_modules` row.