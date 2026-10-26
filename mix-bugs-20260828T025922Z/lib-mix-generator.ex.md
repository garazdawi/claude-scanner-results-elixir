# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/generator.ex
<!-- sha256: beda69a9a756789d490ec6d8d6f88edcb85b1466fac306d7bc04f78ea6618e15 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:16:20Z_

## Summary

`Mix.Generator` is a small helper module for creating/copying files and templates; it is largely sound, but `create_file/3` crashes when handed the `iodata` its own `@spec` permits together with the documented `:format_elixir` option.

## Bugfixes

#### `CSCAN-generator-format-elixir-iodata-crash`
- **Finding ID:** `CSCAN-generator-format-elixir-iodata-crash`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `generator.ex:44`, in `create_file/3` (the `format_elixir` branch)
- **Class:** Contract violation (spec says `iodata`, implementation only accepts `binary` on this path)
- **Impact:** A well-behaved caller who passes `contents` as an iodata list (explicitly allowed by `@spec create_file(Path.t(), iodata, generator_opts)`) *and* sets `format_elixir: true` gets an uncaught `FunctionClauseError` from deep inside `Code.format_string!/2`, not a helpful error. The two documented features — iodata contents and Elixir formatting — are mutually exclusive in practice, contradicting the docs. `copy_file/3` and `copy_template/4` happen to be safe only because they feed binaries in.
- **Evidence:** `verified`. Driving the public API:
  ```
  Mix.Generator.create_file("out.ex", ["defmodule ", "Foo do\nend\n"], format_elixir: true)
  ```
  produced:
  ```
  ** (FunctionClauseError) no function clause matching in Code.format_string!/2
      ...
      def format_string!(string, opts) when is_binary(string) and is_list(opts)
      (mix 1.21.0-dev) lib/mix/generator.ex:44: Mix.Generator.create_file/3
  ```
  `Code.format_string!/2` is guarded by `is_binary(string)`, so any list-shaped iodata fails the clause. The non-`format_elixir` path (`File.write!`) accepts iodata fine, so the crash is specific to this branch.
- **Suggested fix:** Normalize before formatting: `[Code.format_string!(IO.iodata_to_binary(contents)), ?\n]`.

## Risks / caveats
- The `iex>` blocks in `create_file`, `create_directory`, `copy_file`, and `copy_template` look like doctests but are not executed — `test/mix/generator_test.exs` contains no `doctest Mix.Generator`, so their side-effecting/`true` output lines are illustrative only and not a doctest-failure finding.
- `String.to_unsafe_atom("#{name}_template" / "_text")` in `embed_template`/`embed_text` creates atoms from `name`, but these are compile-time macros whose `name` is supplied by the module author (not untrusted runtime input), so atom-table exhaustion is out of scope per the reachability rules.
- I did not exercise every option-combination of the shell-prompt paths (`overwrite?/1,2`) interactively; their logic was reviewed by reading only, but the branches are straightforward and `IO.iodata_to_binary/1` correctly handles iodata in the comparison.

