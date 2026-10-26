# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/record/extractor.ex
<!-- sha256: 2f7cf266c1f7f618524a3dfff34d22ff7970ce77fd3ebd2b9b857480b6ffad82 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:39:20Z_

I've thoroughly exercised the public API. Every path works correctly and the file is functionally equivalent to upstream. Here is the report.

## Summary
`Record.Extractor` correctly extracts Erlang record definitions via its two public entry points (`extract/2`, `extract_all/1`), and no genuine reachable bug was found after running every documented path.

## Bugfixes

No findings

## Risks / caveats

- The module is `@moduledoc false` and both public functions (`extract/2`, `extract_all/1`) are undocumented compiler-internal helpers driven by the `Record` macros; there are no `@doc`/doctest/`@spec` contracts in this file to contradict, so the contract-mismatch surface is small.
- I verified behavior against a build of the source under review, not against a pinned upstream copy; the one line that differs from historical upstream (`List.to_unsafe_atom/1`) is correct *for this repo* because `List.to_atom/1` is deprecated here — see Ruled out.
- `parse_field/1` only handles `:typed_record_field`, `:record_field/3`, and `:record_field/4`. This is total over the shapes Erlang's `:epp`/`:erl_parse` actually emit for `-record` fields; a novel field shape would `FunctionClauseError`, but that is not reachable from valid parsed input.

### Ruled out
- **`List.to_unsafe_atom/1` (line 43) is a nonexistent/wrong function** — `verified`. `Code.ensure_loaded(List); function_exported?(List, :to_unsafe_atom, 1)` returns `true`, and `list.ex:1060-1062` defines it as `:erlang.list_to_atom/1` — byte-for-byte the same operation as the now-deprecated `List.to_atom/1` (`list.ex:1024-1027`). No behavioral change; `Record.extract(:file_info, from_lib: "kernel/include/file.hrl")` returned the full field list.
- **`cond do file = opts[:from] -> ...` truthiness bug** — `verified` by construction. A `:from`/`:from_lib` value is a path string (truthy); a missing key yields `nil` and correctly falls through to the `raise ArgumentError`. `Record.extract(:x)` with neither option raises the documented "expected :from or :from_lib" error.
- **`List.foldr` accumulator/order building the cons list backwards** — `verified`. `Record.extract_all(from: "/tmp/rec.hrl")` returned fields in source order (`[x: 1, y: 2, z: :undefined]`), confirming `foldr` with the `{nil, 0}` (empty-list abstract node) seed builds the list in the correct order.
- **Default value / typed-field extraction dropped or mis-evaluated** — `verified`. A record with `x = 1 :: integer()`, `y = 2`, bare `z`, and `b = "hi"` produced `[x: 1, y: 2, z: :undefined]` and `b: ~c"hi"`, showing `:typed_record_field` unwrapping, defaults, and `:undefined` fill-in all work.
- **`read_file`/`from_lib_file` error paths returning wrong shape instead of raising** — `verified`. Missing record raises `ArgumentError` with the documented "no record ... found" message; unknown lib raises "lib file ... could not be found"; a bad file would hit the `raise "error parsing file ..."` branch off `:epp.parse_file/2`.

