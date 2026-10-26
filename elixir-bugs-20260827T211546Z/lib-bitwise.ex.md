# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/bitwise.ex
<!-- sha256: de6cbf29cf4f04287ddb2c4456a44cb90d3e2ce85f04ceca4207a05935d9d469 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:35:32Z_

All doctests produce the documented values. The file is a thin, correct wrapper over `:erlang` bitwise BIFs.

## Summary
`bitwise.ex` is a thin, correct set of inlined wrappers over Erlang's bitwise BIFs; every doctest reproduces its documented result and no reachable defect was found.

## Bugfixes

No findings

### Ruled out

- **Doctest mismatches across all `@doc` examples** — Ran every documented example (`bnot/1`, `band/2`, `bor/2`, `bxor/2`, `bsl/2`, `bsr/2` and their operators `&&&`, `|||`, `<<<`, `>>>`, including negative-shift and negative-operand cases). Output was `{-3, 1, 1, 1, 11, 11, 10}`, `{4,0,-4,-1}`, `{0,4,-1,-4}` etc., matching each doctest exactly. Basis: `verified`.
- **`@spec integer :: integer` vs implementation** — Each function/operator delegates directly to the corresponding `:erlang` BIF (`bnot/band/bor/bxor/bsl/bsr`), whose contract is integer→integer; the specs describe exactly this. Non-integer input raises `ArithmeticError` as the `@moduledoc` promises, so no crash-on-valid-input or spec divergence exists. Basis: `asserted` (delegation is a one-liner to the BIF; behavior is the BIF's documented contract).
- **`__using__/1` `except:` list construction (`:only_operators` / `:skip_operators`)** — The `:only_operators` branch excepts the six named functions (leaving only operators), and `:skip_operators` excepts the six operator forms (`"~~~"`, `&&&`, `|||`, `"^^^"`, `<<<`, `>>>`), which is the semantically correct complement for each option; the `true ->` fallback excepts nothing. Arities match the exported forms. The macro is `@doc false` and `@deprecated "import Bitwise instead"`, so it is not part of the supported public surface, and its logic is nonetheless correct. Basis: `asserted` (read the two branches against the module's export list; both partitions are complementary and complete).
- **Duplicated deprecated operator defs `~~~/1` and `^^^/2`** — Both are `@doc false`, delegate to `:erlang.bnot`/`:erlang.bxor` identically to their named counterparts, and do not conflict with any other clause (distinct name/arity). No shadowing or missing-clause issue. Basis: `verified` (module compiles and `import Bitwise` succeeds when running the doctest script).

## Risks / caveats
The `unicode`/tokenizer/interpolation threat model supplied as standing context does not apply to this file — `bitwise.ex` performs no parsing, string handling, or untrusted-input processing. Coverage here is complete: the file is 274 lines, entirely delegations plus docs, and all documented behavior was executed against the sandbox build (which emitted only an unrelated latin1-locale VM warning).

