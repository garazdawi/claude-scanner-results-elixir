# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code/fragment.ex
<!-- sha256: 781e647f549b894eaf75d2e65174a1ebfb59ed9caff41b4b55e8ea0a4e1006f5 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:53:36Z_

Confirmed. Here is my report.

# Summary

Extensive fuzzing (700k+ random inputs) and doctest/example verification show `Code.Fragment` is crash-robust, but `container_cursor_to_quoted/2` silently ignores its documented `:file`, `:line`, and `:column` options because they are stripped from `opts` before being read.

# Bugfixes

#### `CSCAN-fragment-cctq-drops-file-line-column-opts`
- **Finding ID** — `CSCAN-fragment-cctq-drops-file-line-column-opts`
- **Severity** — High
- **Confidence** — high
- **Location** — `lib/elixir/lib/code/fragment.ex:1278-1283`, `container_cursor_to_quoted/2`
- **Issue** — Line 1278 does `opts = Keyword.take(opts, [:columns, :token_metadata, :literal_encoder])`, which discards every other key, including `:file`, `:line`, and `:column`. Lines 1281-1283 then read those keys back out of the *already-filtered* `opts` (`Keyword.get(opts, :file, "nofile")`, `:line → 1`, `:column → 1`), so they can never see a caller-supplied value. The tokenizer and `tokens_to_quoted` are therefore always driven with `file = "nofile"`, `line = 1`, `column = 1`.
- **Class** — State / accumulator threading (a binding read after it was rebound to a stripped-down value).
- **Impact** — The documented `:file`, `:line`, and `:column` options are dead. Any tooling (language servers, editors) that passes the cursor's real position in a document — the intended use of this function — gets AST whose `:line`/`:column` metadata and `__cursor__` location are anchored at line 1 / column 1 instead of the requested origin, and parse-error locations/filenames are wrong. Positions are silently incorrect rather than failing loudly.
- **Evidence** — `verified`.
  ```
  Code.Fragment.container_cursor_to_quoted("foo", line: 7, column: 3, columns: true)
  #=> {:ok, {:__cursor__, [line: 1, column: 4], []}}     # line:7 ignored, column origin lost

  Code.string_to_quoted("foo", line: 7, column: 3, columns: true)
  #=> {:ok, {:foo, [line: 7, column: 3], nil}}           # peer API honors both

  Code.Fragment.container_cursor_to_quoted("max(some_value, ", line: 10, column: 5)
  #=> {:ok, {:max, [line: 1], [{:some_value, [line: 1], nil}, {:__cursor__, [line: 1], []}]}}
  ```
  The `line: 10`/`column: 5`/`line: 7`/`column: 3` inputs never affect the output; the peer function `Code.string_to_quoted/2` shows the expected honoring of the same options.
- **Suggested fix** — Read the three options from the full `opts` *before* narrowing it. Move lines 1281-1283 above line 1278 (they only need `opts` after the two `Keyword.pop`s):
  ```elixir
  {preserve_sigils?, opts} = Keyword.pop(opts, :preserve_sigils, false)
  file = Keyword.get(opts, :file, "nofile")
  line = Keyword.get(opts, :line, 1)
  column = Keyword.get(opts, :column, 1)
  opts = Keyword.take(opts, [:columns, :token_metadata, :literal_encoder])
  opts = [check_terminators: {:cursor, preserve_sigils?, []}] ++ opts
  ```

### Ruled out

- **Crash on hostile input to `cursor_context/2`** — Ran ~200k random binaries (length 1-8) over the full punctuation/operator alphabet; zero rescues or throws. Basis: `verified`.
- **Crash on hostile input to `surround_context/3` (single- and multi-line)** — Ran ~500k random inputs with randomized `{line, column}` across the input's line/length range; zero rescues or throws. Guards using `hd(rest)` are total (a raising guard simply fails the clause), so empty-list `hd` does not crash. Basis: `verified`.
- **Crash on hostile input to `container_cursor_to_quoted/2`** — Ran ~200k random fragments × random `trailing_fragment`/`preserve_sigils`/`columns`/`token_metadata` option sets; zero rescues or throws. Basis: `verified`.
- **All published doctests / documented examples** — `lines/1`, `cursor_context/2` (incl. `%URI{` → `:expr`), every `container_cursor_to_quoted/2` example (function call, container, complete-expr collapse, operator, `fn`+`trailing_fragment`, all three `preserve_sigils` cases), and the `surround_context/3` `@`/`do`/`~r`/`&1`/`when`/`key`/`foo` cases all reproduce exactly. Basis: `verified`.
- **Column arithmetic (`build_surround`, `to_multiline_range`)** — Verified `begin`/`end` for ~16 token shapes plus multi-line dot-continuation and `foo(`-continuation cases; end column always equals begin + grapheme length and maps back to the correct source line. Basis: `verified`.
- **Unicode column counting** — `build_surround` uses `:string.length` and `string_reverse_at` uses `:unicode_util.gc`; tested precomposed, combining-mark (`cafe`+U+0301 → span 4, not 5), and Greek identifiers; columns count grapheme clusters correctly. Basis: `verified`.
- **`surround_context/3` returns `:none` for operator-atoms (`:..`, `:+`, `:<>`, `:%`)** while `cursor_context/2` returns `{:unquoted_atom, …}` — this is a longstanding token-shape limitation of the `take_identifier`/`maybe_operator` split (textual atoms like `:when`, `:and`, `:do` work); it is consistent, documented-as-token-based/imprecise, and I could not establish it as unintended, so not reported. Basis: `verified` (behavior), `asserted` (intent).
- **`identifier_to_cursor_context` `n+2`/`n+3` offset literals (lines 344-347)** — token lengths for `:..`, `...`, `:.`, `..` match the `+N` constants and produce correct `surround_context` end columns. Basis: `verified`.

# Risks / caveats

- The reported bug is a functional/contract defect, not a crash or security issue; nothing here corrupts memory or enables an attacker beyond wrong metadata. Its severity is driven by the function's sole purpose (cursor positioning) being defeated for any caller in a multi-line document.
- The reviewed file is byte-identical to the built stdlib in `/opt/elixir` (same `1.21.0-dev` tree), so all behavioral claims were exercised against the exact code under review.
- Absence claims for crashes rest on random fuzzing, not exhaustive enumeration; a pathological non-UTF-8 or extremely long input outside the tested alphabets was not covered, though the `List.Chars`/charlist contract makes such inputs caller-contract violations rather than in-scope defects.

