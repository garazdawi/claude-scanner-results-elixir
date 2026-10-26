# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/diff.ex
<!-- sha256: 77b88df90d6d8b3a497bdefb8fad18f40c5b0df24f260cddc000617bfa700332 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:28:37Z_

## Summary

`ExUnit.Diff` handles the valid inputs its real caller (`ExUnit.Assertions`) produces without defect; the only crash I could provoke requires an AST the Elixir compiler itself rejects, so it is unreachable through the documented flow.

## Bugfixes

No findings

## Risks / caveats

- I exercised `compute/3` across match/`==`/`===` contexts with tuples, lists (proper, improper, keyword, charlist), maps, structs, strings/binaries, bitstrings, numbers (int↔float under `==` vs `===`), functions, pins, guards (bound var true/false, `and`), string-concat patterns, and duplicate vars — all returned correct `equivalent?` results with no crash (verified against the compiled source under review).
- `to_algebra/2` was exercised indirectly only through the structs it receives from `compute`; I did not fuzz it with hand-built diff structs, since those are not part of a documented external contract.
- The pin/pinned-key "RAISED" lines in my probes were test-harness key-construction mistakes (I keyed pins with `{:k, nil}`/`%{x: 6}` instead of the real `var_context` `{name, context}` key), not module defects — the correctly-keyed pin cases (`pin ok`, `pin diff`) pass.

### Ruled out

- **`MatchError` in `split_left_list/3` (diff.ex:476) on a `++` pattern whose left operand is not a proper literal list** — `compute(quote(do: x ++ [3]), [1,2,3], {:match, []})` and `compute(quote(do: ([1] ++ x) ++ [2]), …)` both raise `(MatchError) no match of right hand side value: {:improper, …}`, and `compute` catches only `:undiffable`, so this escapes. **Verified unreachable**: I confirmed with `Code.eval_string/1` that both `x ++ [3] = …` and `([1] ++ x) ++ [2] = …` fail to compile — Elixir rejects them with *"invalid argument for ++ operator inside a match, expected a literal proper list"*. Since `compute` is fed only ASTs of assertions that the compiler already accepted, no valid match pattern reaches line 476 with a non-empty improper-left. It is a latent partial function but not reachable through the documented assertion flow (folder threat model: inputs are compiled test-author code, not adversarial ASTs).
- **`safe_eval`/`safe_inspect`/`load_struct` catching only `rescue` (not `throw`/`exit`)** — reachable only for struct keyword literals guarded by `Macro.quoted_literal?`, whose evaluation cannot `throw`/`exit`; verified struct diffing (`URI` mismatch) returns cleanly.
- **Grapheme vs byte length in `diff_string_concat`/`parse_string`/`next_concat_result`** — `parse_string` (`String.length`), `String.split_at`, and `next_concat_result` (`String.length` + `String.split_at`) all operate in graphemes consistently; the `"ab" <> rest` binary-concat pattern verified `eq=true`. No byte/grapheme mismatch.
- **`diff_map_by_key` sorting mixed-type AST/value pairs** (`Enum.sort`) — Erlang total term ordering makes this total; nested and keyword map diffs verified without raising.

