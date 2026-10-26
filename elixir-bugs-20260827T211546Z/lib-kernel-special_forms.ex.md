# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/special_forms.ex
<!-- sha256: db58059f0c8c36cea4b28ae9f56c5412f7d6b02379563fa93b771608f240cbdd -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:30:29Z_

All 98 doctests pass, and the file under review is byte-for-byte identical to the released `/opt/elixir` copy. This is a documentation-only module: every macro body calls `error!` by design (the real behavior lives in the compiler), so the only executable contracts are the doctests, which I ran.

## Summary
No genuine, reachable bugs were found: every macro in this file intentionally raises via `error!`, and all 98 executable doctests pass when run against a from-source build.

## Bugfixes

No findings

### Ruled out

- **Doctest failures across the whole module** — I ran `doctest Kernel.SpecialForms` (which executes every `iex>` example in this exact file) under the source-built toolchain: `Result: 98 passed`. Search space: all 98 doctests in the file; none failed. Basis: `verified`.
- **`case/2` doc example with the stray `...> value` block (lines 1937–1943)** — the odd continuation formatting after the blank line parses and passes as part of the 98 doctests (it is a `** (CompileError)` expectation, matched). Basis: `verified`.
- **`with/1` `double_width` example (lines 1619–1628) returning `{:ok, 300}` and `width` staying `nil`** — included in the passing doctest run; arithmetic (10·2·15=300) and hygiene both hold. Basis: `verified`.
- **Bitstring/`for` comprehension examples (`for <<c <- ...>>`, `uniq:`, `into:`, signed/endianness sections)** — all are among the 98 passing doctests; values (`"helloworld"`, `"ABC"`, `256`/`1` endianness, `156`/`-100` sign) reproduce. Basis: `verified`.
- **`try/1` else-clause compiler warning at line 2100 ("clause cannot match")** — this is an intentional illustrative example whose doctest still passes; it is a type-system warning about a deliberately-redundant `_ ->` fallback in documentation, not a defect in this file's behavior. Basis: `verified`.
- **`@spec`/contract mismatches** — the module declares no `@spec`s; each macro's sole documented contract is "raises when invoked directly," which every body satisfies (`:erlang.error(RuntimeError.exception(...))`). Basis: `verified` (read all 2403 lines; no `@spec` present).

## Risks / caveats
This file carries no runtime logic of its own — the semantics of `case`, `with`, `try`, `for`, bitstrings, etc. are implemented in the compiler front-end (`elixir_expand`, `elixir_tokenizer`, and friends), so any real behavioral bug would surface there, not here. My verification confirms the *documentation contracts* in this file match the installed compiler's behavior; it does not re-audit those compiler stages. The build emits a latin1-encoding warning (`ELIXIR_ERL_OPTIONS="+fnu"` worked around it); doctests involving non-ASCII would be worth re-checking under a proper UTF-8 locale, though none of this file's examples depend on non-ASCII identifiers.

