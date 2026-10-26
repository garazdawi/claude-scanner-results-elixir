# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code/formatter.ex
<!-- sha256: b222f82dced7a0e29a9ec451749bc4041772a88999a3c3277ecd1ec6dd993e4e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:52:11Z_

I've done extensive behavioral testing against the public path (`Code.format_string!/2`, which drives this module) across migrations, escaping, numeric formatting, operator precedence, comments, sigils, and bitstrings. Everything I tried produced correct, idempotent output. Here is the report.

## Summary
After reading the full file and exercising its public entry point (`Code.format_string!/2` → `Code.Formatter.to_algebra/2`) across migration options, escaping, numeric tokens, operator nesting, comments, sigils, and bitstrings, I found no genuine, reachable bug.

## Bugfixes

No findings

### Ruled out

- **`float_to_algebra/2` (line 1725) `[int_part, decimal_part] = :binary.split(text, ".")` MatchError when the token has no `.`** — The float `:token` is the verbatim source text of a float literal, and the Elixir grammar requires `digits "." digits`, so a `.`-free token cannot arise from valid input; producing it requires hand-building AST that violates the documented token contract (out of scope). Verified that legitimate exponent/underscore forms format correctly: `1.0e10`, `1000000.0e10 → 1_000_000.0e10`, `1.0E10 → 1.0e10`. Basis: `verified` for reachable inputs; grammar-`asserted` for the unreachable one.
- **`migrate_unless` rewrites change semantics** — Checked plain, boolean-operator, comparison, and pipe conditions. `unless foo → if !foo`; `unless a and b → if !(a and b)` (operand correctly parenthesized); `unless a == b → if a != b`; `x |> foo() |> unless(do: y) → x |> foo() |> Kernel.!() |> if(do: y)` (equivalent to `if !foo(x), do: y`). All semantically equivalent and idempotent. Basis: `verified`.
- **`migrate_atom_interpolations` emits `String.to_unsafe_atom/1` which might not exist / be non-equivalent** — Confirmed `String.to_unsafe_atom/1` is exported in this build (`function_exported?/3 → true`, defined at `string.ex:3040`) and is the non-deprecated unsafe-atom constructor; `:"#{x}" → String.to_unsafe_atom("#{x}")` is equivalent. Basis: `verified`.
- **`migrate_bitstring_modifiers` normalization (`bitstring_spec_normalize_empty_args/1`) adds/omits parens incorrectly** — Known specs keep no parens (`binary`, `integer-size(8)`, `size(8)-unit(2)`), unknown modifiers gain them (`custom-integer → custom()-integer`), `_` and `*`/`-` combinations are handled; all idempotent. Basis: `verified`.
- **`insert_underscores/1` / `do_insert_underscores/2` digit grouping off-by-one** — `123456789 → 123_456_789`, `100000.000123 → 100_000.000123` (decimal part untouched, correct), already-underscored and <6-digit tokens left as-is. Basis: `verified`.
- **Operator parenthesization in `binary_operand_to_algebra/7` (`@required_parens_*` tables)** — `a -- b -- c → a -- (b -- c)`, `a --- b --- c → a --- (b --- c)`, `(a || b) && c` preserved, `a in b in c` preserved (round-trips to same AST via left-associativity of `in`). Basis: `verified`.
- **Comment gathering / `format_comment_text/1` / trailing-newline logic** — Leading comment before a pipe, interior call-argument comments, and `##`/`#!`/`#` normalization all format and round-trip correctly. Basis: `verified`.
- **`atom_to_algebra(:\\, …)` and quoted-atom escaping** — `:\\` and `:"\\"` retain their distinct delimiters and round-trip. Basis: `verified`.
- **`split_last/1` MatchError on `[]`** — Every caller either handles the empty-arg case separately (the `call_args_to_algebra([], …)` head at line 1196) or is guaranteed a non-empty list (`when` clauses have ≥2 args). No public path reaches `split_last([])`. Basis: `asserted` (call-site enumeration within this file).

## Risks / caveats
- The module is `@moduledoc false`; I treated `Code.format_string!/2` as the real documented public path and drove all tests through it. Bugs reachable only by passing hand-constructed AST that violates the formatter's token/metadata contract (e.g. an integer/float `__block__` node lacking `:token`, a float token without `.`) are out of scope by the stated rules and were not pursued as findings.
- My search was behavioral/differential (crash + idempotency + semantic spot-checks on the newer migration features), not exhaustive over all layout-width branches. A purely cosmetic layout defect that still re-parses to the same AST and is idempotent would not be caught by this method; none of the classes in scope (correctness, crashes, contract, semantics) surfaced anything.
- `format_to_string/1` (used by `capture_to_algebra/2` and `bitstring_wrap_parens/3`) renders sub-documents at `:infinity` width to inspect their leading/trailing bytes; this is an O(n) re-render per call but a performance characteristic, not a correctness bug, so it is not reported.

