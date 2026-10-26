# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir.erl
<!-- sha256: 7e48fffe56aa720347d802849247e8648c069e50c0ee47e5a8666fcd72071c7d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T05:57:44Z_

I've now verified the cross-file contracts (tokenizer error/ok tuple arities, parser location shapes via `return_error/3`). The file is a faithful copy of upstream Elixir compiler front-end code. Here is my report.

# Bug-hunting review — `elixir.erl`

## Summary
After tracing every documented public entry point and verifying the tokenizer/parser return-shape contracts in neighbouring modules, I found no correctness, crash, resource, or contract defects reachable through this module's public API.

## Bugfixes

No findings.

The whole file was read (`elixir.erl:1-543`). The candidate crash-shapes I chased all dissolved on cross-file verification:

- **`parser_location/1` (lines 486-498) on a bare-integer location** — I hypothesised a `badarg` in `lists:keyfind/3` if `tokens_to_quoted` handed it an integer. Verified against `elixir_parser.yrl:1183-1184`: the custom `return_error/3` rewrites `{Line,Column,_}` into a meta list `[{line,L},{column,C}]`, and yecc's default errors pass the token's `{Line,Column,_}` tuple. Both shapes are covered by the two `parser_location` clauses; an integer location never occurs. Not a bug.
- **`string_to_tokens/5` error clause (line 460)** — the 5-tuple `{error, Info, _Rest, _Warnings, _SoFar}` exactly matches `elixir_tokenizer:error/4`'s return `{error, Reason, Rest, Warnings, Tokens}` (`elixir_tokenizer.erl:1917-1918`), and the 6-tuple ok clause matches the tokenizer's `{ok, Line, Column, Warnings, Tokens, Terminators}` returns (`elixir_tokenizer.erl:159,179`). No shape mismatch.
- **`tokens_to_quoted/3` `try...of` clauses (lines 468-473)** — the two message shapes `[{Prefix,Suffix},Token]` and `[Error,Token]` cover both Elixir's custom `return_error/3` (2-element `[Message, Token]` list) and yecc's default syntax-error message. No third shape is produced by the parser.

## Risks / caveats
- The dismissals for `parser_location/1`, `string_to_tokens/5`, and `tokens_to_quoted/3` depend on `elixir_parser.yrl` and `elixir_tokenizer.erl` continuing to emit exactly the location/error shapes verified above. A future change to `return_error/3` (to pass a raw integer or binary location) or to `elixir_tokenizer:error/4`'s tuple arity would reintroduce a `badarg`/`try_clause` crash in `string_to_quoted`. These are the only out-of-file contracts this file leans on.
- `env_for_eval/1`'s first clause is guarded by `map_size(Env) == 15` (line 199). This silently depends on the `Macro.Env` struct having exactly 15 fields; if that struct gains or loses a field without this literal being updated, a valid env would fall through to a `function_clause`. It is not a bug today (the struct is 15 fields in this same tree) but it is a brittle coupling worth noting.

### Ruled out

**Operator / precedence typo**
- Reason: The one compound boolean guard (`env_for_eval/1` map clause `map_size == 15`), the OTP-28 warning condition (`parse_otp_release`, line 132: `Num == 28 andalso ... andalso not erlang:function_exported(...)`), the literal guard in `eval_forms` (line 323: `Literal == atom; Literal == float; Literal == integer`), and `emit_warnings`'s `andalso` chain all parse with the intended precedence when read with explicit parentheses.
- Axes examined: `;`-vs-`,` in the literal guard, `andalso`/`not` binding in the OTP check, `==` vs `=:=` on integer/atom operands | Not examined: none
- Basis: verified

**Missing clause / catch-all**
- Reason: `drop_common/3` (lines 435-439) covers empty/non-empty first arg and empty/non-empty second arg; `parser_location/1`, `to_binary/1`, and the tokenizer/parser return-shape matches were each verified against the actual producing code in `elixir_tokenizer.erl` and `elixir_parser.yrl`.
- Axes examined: empty vs non-empty stacktrace lists, integer/tuple/meta-list location, list/atom message elements | Not examined: none
- Basis: verified

**Wrong atom / wrong symbol**
- Reason: Config keys (lines 82-102), env record field updates (`context`, `macro_aliases`, `versioned_vars`, `tracers`, `lexical_tracker`), and the process-dictionary keys (`{elixir, dbg_callback}`, `?elixir_eval_env`) are used consistently for both read and write within the file.
- Axes examined: config key spelling vs consumers, `Env#{...:=...}` field names against the `map_size==15` struct, PD key symmetry across put/get/erase | Not examined: none
- Basis: verified

**State / accumulator threading error**
- Reason: `eval_forms` threads `ErlS→NewErlS`, `ExS→NewExS`, `E→NewE` through `quoted_to_erl` and uses the post-eval values in `dump_binding`; the `PreviousDbg`/`PreviousEvalEnv` save-restore in `eval_forms`/`erl_eval` uses the captured pre-value in the `after` block.
- Axes examined: nested-eval save/restore (undefined vs defined previous value), Prune true/false branches returning correct binding tuple | Not examined: none
- Basis: verified

**Off-by-one / wrong arithmetic**
- Reason: The only arithmetic is `length(Binding)` for `PruneBefore` (line 331, with `-1` sentinel when not pruning) and `map_size == 15`; both are used as intended by `elixir_erl_var:dump_binding`.
- Axes examined: empty binding (length 0), non-empty binding, Prune true/false sentinel | Not examined: none
- Basis: verified

**Contract violation**
- Reason: Each public function returns its documented shape — `string_to_tokens` returns `{ok,_,_}|{error,_}`, `string_to_quoted` returns `{ok,Forms}|{error,_}`, `'string_to_quoted!'` returns `Forms` or raises via `elixir_errors:parse_error`, `eval_forms` returns `{Value,Binding,Env}`.
- Axes examined: ok vs error return of tokenize/parse, literal-tree vs general-tree eval return, bang-vs-non-bang variants | Not examined: none
- Basis: verified

**Crash on stray / unexpected input**
- Reason: The public parse/token entry points delegate malformed input to the tokenizer/parser which return `{error,...}` tuples (verified shapes match). `eval_external_handler`'s catch is `Kind:Reason:Stacktrace` (catches all classes). No `receive` or gen_server callback lives in this file.
- Axes examined: malformed source bytes → tokenizer error tuple, syntax error → parser error tuple, exception class throw/error/exit in external handler | Not examined: none
- Basis: verified

**Resource leak on error path**
- Reason: `start/2` deletes the config table on `elixir_sup:start_link/0` failure (lines 110-112); `eval_forms`/`erl_eval` restore the process-dictionary keys in `after` blocks regardless of exception.
- Axes examined: sup start failure path, exception during eval (PD restore), nested eval (previous value preservation) | Not examined: none
- Basis: verified

**Silent corruption**
- Reason: `string_to_tokens` returns `lists:reverse(Tokens, Terminators)` matching the tokenizer's reversed-accumulator convention; `emit_warnings` reverses warnings before emitting (chronological order). No truncation or accumulator-drop paths.
- Axes examined: token order (reverse+terminators), warning emission order, empty-warning short-circuit to `false` | Not examined: none
- Basis: verified

**Infinite loop / hang**
- Reason: No `receive`, no gen_server `handle_call`, and no unbounded recursion in this file; `drop_common` strictly decreases its first argument's length each step.
- Axes examined: recursion decrement in drop_common, absence of receive/gen callbacks | Not examined: none
- Basis: verified

**Quadratic-where-linear**
- Reason: No `++` in a recursive accumulator, no `length/1` in a guard, no per-iteration `lists:member`; `Bottom ++ Reversed` (line 426) is a one-shot concatenation on short stacktrace lists, not a loop.
- Axes examined: accumulator patterns, guard-side length calls, one-shot vs looped concat | Not examined: none
- Basis: verified

**Repeated work**
- Reason: No pattern compilation, BEAM scanning, or recomputation inside a loop; `preload_common_modules` and config setup run once at application start.
- Axes examined: compile-pattern/re-compile in loops, per-call rebuilds | Not examined: none
- Basis: verified

**Dead / unreachable code**
- Reason: Every clause is reachable from a public entry — both `env_for_eval` clauses (map vs list), both eval_forms branches (literal vs general), all `drop_common` heads.
- Axes examined: env-struct vs opts-list dispatch, literal vs general eval, stacktrace-prune branches | Not examined: none
- Basis: verified

**Documented behaviour mismatch**
- Reason: The module's only documented claims are the `-spec`-less function contracts and the moduledoc note ("private to the Elixir compiler"); the deprecation warnings in `env_for_eval` (`:aliases`/`:requires`/`:functions`/`:macros`/`:lexical_tracker`) each fire on the corresponding option as their prose states.
- Axes examined: each deprecated option present vs absent, warning text vs option key | Not examined: none
- Basis: verified

**Race condition / shared mutable state**
- Reason: The process-dictionary keys (`dbg_callback`, `?elixir_eval_env`) are per-process and save/restored around nested calls; config writes go through `elixir_config` (single gen_server), not touched racily here.
- Axes examined: nested same-process eval (PD isolation), config init ordering | Not examined: cross-node config singleton (out of file, covered by `elixir_config` report)
- Basis: verified

**Wrong scope / order of operations**
- Reason: `start/2` sets `elixir_config:static/1` before `elixir_config:new/1` and starts the supervisor last, deleting the table only on failure; `eval_forms` captures `PreviousDbg` before overwriting it.
- Axes examined: config static-before-new ordering, save-before-overwrite in eval | Not examined: none
- Basis: verified

**Encoding / unicode mishandling**
- Reason: `to_binary/1` routes lists through `elixir_utils:characters_to_binary` (unicode-aware) and atoms through `atom_to_binary`; `check_file_encoding` warns on latin1; `io:setopts(standard_io, [binary])` is set at start.
- Axes examined: list (charlist) vs atom message element, latin1 vs utf8 native encoding | Not examined: binary message element (does not occur — verified parser emits only charlist/atom message parts in `elixir_parser.yrl`)
- Basis: verified

**Unclassified**
- Reason: The `env_for_eval/1` `map_size == 15` brittleness (noted in caveats) is a maintenance coupling, not a defect on current inputs; no other file-specific anomaly surfaced.
- Axes examined: struct-size literal vs actual Macro.Env field count in this tree | Not examined: none
- Basis: verified

### Did I verify this?

All ruled-out entries are marked `verified`: for each I read the relevant clauses in `elixir.erl` and, where the dismissal depended on out-of-file behaviour, traced the producing code — `elixir_tokenizer.erl:159,179,1917-1918` (ok/error tuple arities) and `elixir_parser.yrl:1183-1184,718,746` (`return_error/3` location and message shapes). The `Not examined` lists all read `none` except two entries whose remaining axis is an out-of-file contract explicitly delegated to the neighbouring module's report:

- **Race condition** — the config singleton's cross-node behaviour lives in `elixir_config.erl`; `elixir.erl` only initialises it once at start, which I traced. The in-file PD isolation is fully verified.
- **Encoding** — the "binary message element" axis cannot arise because I verified `elixir_parser.yrl`'s `return_error/3` only ever passes charlists and atoms as message/token parts; no binary reaches `to_binary/1` from the parser path.

No `asserted` dismissals remain, so no `Depends on:` lines are outstanding beyond the two cross-file contracts named in Risks / caveats.

