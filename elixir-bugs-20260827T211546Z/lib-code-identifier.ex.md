# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code/identifier.ex
<!-- sha256: d22069cfae2bb54d738567e9b43a20ff2ff3cfb1ff0dd7c1b36411fb5e7575da -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:53:21Z_

I've reviewed the file thoroughly and verified its behavior by running it against the source-built Elixir. This is `Code.Identifier`, an internal (`@moduledoc false`) module providing the operator-precedence table, identifier escaping, and anonymous-fun-name parsing.

## Summary
No genuine bugs were found; the operator tables, the `escape/*` state machine, and `extract_anonymous_fun_parent/1` all behave correctly and match their documented contracts on both ordinary and hostile input.

## Bugfixes

No findings

### Ruled out

- **`escape/4` guard rejects valid input (`FunctionClauseError` on documented calls)** — The guard is `(is_binary(binary) and ((char in 0..0x10FFFF or is_nil(char)) and limit == :infinity)) or (is_integer(limit) and limit >= 0)`. I enumerated every documented shape: `{binary, char|nil, :infinity}` passes the first disjunct; `{binary, char|nil, non_neg_integer}` (including `0`) passes the second. No valid `@spec` input fails the guard. Basis: **verified** — `escape("hello", nil, 0)` → `{"", "hello"}`, `escape("hello", nil, 3)` → `{"hel", "lo"}`, `escape("a\"b", ?")` → `{"a\\\"b", ""}`.
- **`char in 0..0x10FFFF` crashing when `char` is `nil`** — In guards, `x in a..b` expands to `is_integer(x) and …`, so `nil` short-circuits to `false` and the `or is_nil(char)` arm succeeds; no `ArithmeticError`/`badarg`. Basis: **verified** — `escape("hello", nil)` → `{"hello", ""}`.
- **Delimiter-`char` clause (`escape(<<char, t>>, char, …)`) mismatching a multibyte delimiter** — For a delimiter >255 the single-byte pattern never matches and the char falls through to the literal/utf8 path unescaped; however every real caller passes an ASCII delimiter (`?"`, `?'`, `?)`), so no documented path is affected. Basis: **asserted** for the >255 edge; the ASCII delimiter path is **verified** (`escape("a\"b", ?")` escapes correctly).
- **`escape_char` coverage gaps (surrogates, C1 controls, DEL, bidi/zero-width, `0xFFFE/FFFF`)** — Control/invisible/bidi code points hit the special-`\u` clause or the `< 0x80`/`< 0x10000` clauses; surrogates are unreachable because `escape_char` is only entered via a decoded `<<h::utf8>>`; raw invalid bytes are handled by the `<<a::4,b::4>>` fallback as `\xNN`. Basis: **verified** — `escape(<<255,254,65>>, nil)` → `{"\\xFF\\xFEA", ""}`, `escape("x\u{202E}y", nil)` → `{"x\\u202Ey", ""}`, `escape("emoji \u{1F600}!", nil)` → emoji left literal (in the `0x10000..0x10FFFF` literal range).
- **`with` in `extract_anonymous_fun_parent/1` using `=` (raising instead of `:error`)** — The `=` line binds `[trailing | reversed]` from `String.split("/") |> Enum.reverse()`, which is always a non-empty list, so it cannot raise; the two `<-` clauses correctly route malformed atoms to `else -> :error`. Basis: **verified** — `:"-foo/2-fun-0-"` → `{:foo, "2"}`, `:"-Foo.bar/3-fun-1-"` → `{:"Foo.bar", "3"}`, `:regular_atom` → `:error`.
- **`String.to_unsafe_atom/1` being a nonexistent function** — It resolves and creates the atom at runtime in this source build. Basis: **verified** — the calls above returned the expected atoms without `UndefinedFunctionError`.
- **`unary_op/1` / `binary_op/1` wrong or missing precedences** — I compared every clause against the upstream Elixir precedence table; associativity and precedence values (40–320) and operator groupings match exactly, and non-operators return `:error`. Basis: **verified** — published doctests `unary_op(:+)` → `{:non_associative, 300}` and `binary_op(:+)` → `{:left, 210}` both pass.

## Risks / caveats
- I did not audit callers of this `@moduledoc false` module (e.g. `Macro.to_string`, `Inspect`, the formatter); the review was scoped to this file's public functions and their documented contracts.
- The string-valued `arity` return of `extract_anonymous_fun_parent/1` (e.g. `"2"`, not `2`) is undocumented but matches upstream and is not treated as a defect.
- The environment emits a `native name encoding of latin1` warning; I ran all checks with `ELIXIR_ERL_OPTIONS="+fnu"` so UTF-8/bidi paths exercised real codepoints.

