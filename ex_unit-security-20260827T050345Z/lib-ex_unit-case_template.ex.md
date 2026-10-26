# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/case_template.ex
<!-- sha256: aa6732badc6576e758d7a0f9f1d210878bfde5047cd0d695366c7da3bb98611d -->

_Analysed on 2026-08-27T05:15:47Z_

I've completed the trace. Both cross-file consumers of caller data are benign, and nothing auto-executes the doc examples.

- `ExUnit.Case.__keys__/1` is just `Keyword.take(opts, @keys)` — an allow-list filter, no atom creation or dispatch sink.
- No `doctest ExUnit.CaseTemplate` exists anywhere in the tree, so the `@moduledoc`/`@doc` code examples are never compiled or run.

Here is my report.

## Summary

`ExUnit.CaseTemplate` is a pure compile-time metaprogramming module: `__using__/1` and the `using/2` macro generate the `__using__` boilerplate that lets a test-support module act as a reusable case template, and `__proxy__/2` builds the quoted `use ExUnit.Case` + `setup`/`setup_all` injection. Every input it touches — the module name (`__MODULE__`), the `opts` passed to `use MyCase, …`, and the `var`/`block` of `using/2` — is trusted developer source expanded at compile time; there is no network, file, deserialization, or atom-creation sink, and no untrusted-data path, so I found nothing meaningful to report.

## Security

No findings

### Ruled out

- **`unquote(module)` / `unquote(opts)` / `unquote(block)` / `unquote(var)` (code execution, unquote hygiene, lines 76, 79, 90, 93–97, 155–159)** — All unquoted values are AST written by the test author in their own source: `module` is the compile-time-constant `__MODULE__`, `opts` is the literal argument list to `use MyCase, …`, and `var`/`block` are the pattern and `quote` body the developer wrote in `using/2`. These are the same trust domain (the developer's own test code) evaluated at compile time; nothing forwards network/disk/dependency-controlled bytes into them, so there is no injection crossing a trust boundary. This is standard, correct macro hygiene (`var` injected unhygienically to bind the caller's variable, `opts` hygienic).
- **`ExUnit.Case.__keys__(unquote(opts))` (atom creation / dispatch, line 90)** — Traced cross-file: `__keys__/1` is `Keyword.take(opts, @keys)`, a fixed allow-list filter with no `String.to_atom`, `Module.concat`, or dynamic dispatch. `opts` itself is trusted source. No sink.
- **Compile-time execution (whole file)** — Yes, all of this runs during `mix compile`, but the only inputs are the test author's own source and the `opts` they hand-write into `use`. A dependency that ships a case template runs its *own* macro code, which the developer opted into by depending on it; this file neither reads external files/env/network at build time nor amplifies a dependency's reach. No build-machine attack surface originates here.
- **`defmacro __using__(unquote(var) = opts)` pattern match (line 155)** — The `var` default `quote(do: _)` matches anything; a developer-supplied pattern that fails to match their own `opts` raises a compile-time `MatchError` in their own build. Self-inflicted, not a reachable defensive gate on untrusted input; not "dead defensive code" in the security sense.
- **Moduledoc/doc code examples (lines 36–64, 111–145)** — These are indented `defmodule` snippets, not `iex>` doctest prompts, and no module runs `doctest ExUnit.CaseTemplate`, so `ExUnit.DocTest` never compiles or evaluates them. No compile/eval sink.

## Risks / caveats

- I treated `__proxy__/2` as effectively internal: it is `def` (exported) but `@doc false`, and is only ever invoked from the macro-generated `__using__`. Even considered as a reachable entry point, it only assembles quoted AST from a module atom and an opts term — no sink — so the distinction does not change the verdict.
- Scope boundary respected: the atom-table and `binary_to_term` concerns called out in the folder threat model live in `ExUnit.Filters` and `ExUnit.FailuresManifest`, not in this file. `opts` here flows only into `ExUnit.Case.__keys__` (allow-list `Keyword.take`), which I confirmed by reading `case.ex:368`; I did not audit the remainder of `ExUnit.Case.__using__`/`__register__`, as those are separate files outside this review.
- Conclusion rests on the standard ExUnit model that `use` arguments are compile-time literals authored by the developer. If a project deliberately funnels externally-controlled data into a `use MyCase, …` option list, that is a self-inflicted pattern in the consuming project, not a defect this file introduces.

