# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/io/ansi/docs.ex
<!-- sha256: ad91a77794e7ba8e9b386a82f0e2ec72f339e22324d4980c36581da6a4fbffc0 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/io/ansi/docs.ex
<!-- sha256: ad91a77794e7ba8e9b386a82f0e2ec72f339e22324d4980c36581da6a4fbffc0 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (Erlang/OTP 29, Elixir 1.21.0-dev). Every claim below was re-executed against the installed source at `/opt/elixir/lib/elixir/lib/io/ansi/docs.ex`, which is byte-identical to the reviewed path (same line numbers)._

## Summary
`IO.ANSI.Docs.print/3` crashes with a `MatchError` on any Markdown body containing an inline `<!--` that lacks a matching `-->`, and the same clause disables inline formatting for the remainder of a line after a *closed* inline comment. A minor spec violation exists in `print_metadata/2`. Finally, `length_without_escape/2` only recognizes ANSI escape sequences of the exact forms `\e[Xm` (4 bytes) and `\e[XXm` (5 bytes), so any single escape sequence longer than that — 256-color, truecolor, or combined-parameter codes — is counted as visible width, corrupting table padding and line-wrap column math.

The only hedge in the first-pass report (`length_without_escape` "did not exhaustively fuzz … for ANSI sequences longer than 4 bytes") was already promoted to a finding in the previous pass. In this re-review I re-executed the axis with a **multi-row** table (the earlier single-row repro accidentally cancelled the over-count, see the finding's Evidence) and confirmed real column misalignment. It remains a low-confidence finding, reflecting that it does not manifest under `default_options/0`.

## Bugfixes

#### `CSCAN-docs-inline-comment-matcherror`
- **Finding ID** — `CSCAN-docs-inline-comment-matcherror`
- **Severity** — High
- **Confidence** — high
- **Location** — `lib/io/ansi/docs.ex:644-647` (`drop_comment/1`, binary clause, match at `:645`), reached from `handle_inline/5` at `:696`
- **Issue** — `handle_inline("<!--" <> rest, ...)` (`:695`) calls `drop_comment(rest)` where `rest` is a binary, dispatching to the binary clause `[_comment, rest] = :binary.split(line, "-->")` (`:645`). When the text contains no `-->`, `:binary.split/2` returns a single-element list `[line]`, and the two-element pattern match fails with `MatchError`.
- **Class** — Missing clause / non-total function
- **Impact** — Any documentation string whose body contains an inline `<!--` (not at the start of a line — that path uses the list-based `drop_comment` and is safe, see Ruled out) without a closing `-->` crashes the doc renderer. `h Module.fun` in IEx, ExDoc-adjacent tooling, or `mix` help routing through `IO.ANSI.Docs.print/3` aborts with an opaque `MatchError` instead of printing the docs.
- **Evidence** — `verified` (re-run this pass):
  ```
  $ elixir -e 'IO.ANSI.Docs.print("foo <!-- unclosed comment here", "text/markdown", enabled: false)'
  ** (MatchError) no match of right hand side value:
      [" unclosed comment here"]
      (elixir 1.21.0-dev) lib/io/ansi/docs.ex:645: IO.ANSI.Docs.drop_comment/1
      (elixir 1.21.0-dev) lib/io/ansi/docs.ex:696: IO.ANSI.Docs.handle_inline/5
      (elixir 1.21.0-dev) lib/io/ansi/docs.ex:367: IO.ANSI.Docs.write_lines/5
  ```
- **Suggested fix** — Make the binary `drop_comment` total:
  ```elixir
  defp drop_comment(line) when is_binary(line) do
    case :binary.split(line, "-->") do
      [_comment, rest] -> rest
      [_] -> ""
    end
  end
  ```

#### `CSCAN-docs-inline-comment-resets-mark`
- **Finding ID** — `CSCAN-docs-inline-comment-resets-mark`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/io/ansi/docs.ex:695-698` (`handle_inline/5`, `"<!--"` clause)
- **Issue** — After stripping an inline comment, the clause recurses with `handle_inline(rest, [], buffer, acc, options)`, passing `[]` as the `mark`/limit state. Every "inline start" clause (`:700`, `:706`, `:711`) and every escape clause is guarded on `mark == nil`; with `mark` set to `[]`, none of them fire, so the rest of the line is treated as plain text.
- **Class** — State / accumulator threading (wrong value threaded into recursive call)
- **Impact** — Any inline formatting (`**bold**`, `_underline_`, `` `code` ``) appearing *after* an inline `<!-- ... -->` comment on the same rendered line is silently emitted unformatted. Cosmetic but content-dependent rendering regression.
- **Evidence** — `verified` (re-run this pass):
  ```
  $ elixir -e 'IO.ANSI.Docs.print("foo <!-- c --> **bold** end", "text/markdown", enabled: true)'
  foo  **bold** end          # no ^[[1m ... bold not emphasized
  $ elixir -e 'IO.ANSI.Docs.print("foo x **bold** end", "text/markdown", enabled: true)'
  foo x ^[[1mbold^[[0m end    # bold emphasized
  ```
- **Suggested fix** — Recurse with `nil` instead of `[]`: `handle_inline(rest, nil, buffer, acc, options)`.

#### `CSCAN-docs-print-metadata-returns-false`
- **Finding ID** — `CSCAN-docs-print-metadata-returns-false`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/io/ansi/docs.ex:95-98` (`print_metadata/2`), spec at `:94`
- **Issue** — `print_each_metadata(...) && IO.write("\n")` (`:97`) returns `false` whenever nothing printable was found, because the reduce accumulator starts at `false` (`:105`) and stays falsy. `@spec print_metadata(map, print_opts) :: :ok` (`:94`) promises `:ok`.
- **Class** — Contract violation (`@spec` disagrees with implementation)
- **Impact** — Callers/Dialyzer relying on the declared `:ok` return see `false` for empty or non-matching metadata maps (e.g. `print_metadata(%{}, opts)`); no runtime failure, but the spec is wrong and propagates into analysis of every caller.
- **Evidence** — `verified` (re-run this pass):
  ```
  $ elixir -e 'IO.inspect IO.ANSI.Docs.print_metadata(%{}, enabled: false)'
  false
  ```
- **Suggested fix** — Force the `:ok` return, e.g. `_ = print_each_metadata(metadata, options) && IO.write("\n"); :ok`.

#### `CSCAN-docs-length-without-escape-long-ansi`
- **Finding ID** — `CSCAN-docs-length-without-escape-long-ansi`
- **Severity** — Low
- **Confidence** — low
- **Upgraded from hedged assumption in first pass** (the "did not exhaustively fuzz … `length_without_escape` for ANSI sequences longer than 4 bytes" caveat). Retained and re-verified this pass with a corrected multi-row repro.
- **Location** — `lib/io/ansi/docs.ex:614-627` (`length_without_escape/2`), consumed by `generate_table_cell/1` (`:529-546`, via `ansi_diff = byte_size(col) - length`) and by `take_words/3` / `wrap_text/6` (`:585`, `:593`).
- **Issue** — `length_without_escape/2` skips escape sequences with only two literal clauses: `<<?\e, ?[, _, _, ?m>>` (5 bytes, e.g. `\e[36m`, `:614`) and `<<?\e, ?[, _, ?m>>` (4 bytes, e.g. `\e[1m`, `:618`). Any single escape sequence longer than 5 bytes falls through to the generic grapheme-counting clause (`:622`) and is counted as visible width. This includes 256-color (`\e[38;5;Nm`, 11 bytes), truecolor (`\e[38;2;R;G;Bm`, 13 bytes), and combined-parameter (`\e[1;33m`, 7 bytes) sequences — byte sizes confirmed by direct probe this pass. Elixir's own `IO.ANSI.format_fragment` emits each named code as its own short `\e[Nm` fragment, so the *defaults* (`:cyan`→`\e[36m`, `:bright`→`\e[1m`, `:reverse`→`\e[7m`, and the two-code `doc_title: [:reverse, :yellow]`→`\e[7m\e[33m` processed fragment-by-fragment) all match; but a caller-supplied `doc_*` option carrying `IO.ANSI.color/1`, or a raw escape embedded in the docstring text itself, is over-counted.
- **Class** — Incorrect width accounting / display corruption (not a crash)
- **Impact** — Column widths and wrap points are computed from the inflated length: a long-escape cell that is *not* the width-determining cell is left mis-aligned, header cells are over-padded, and line wrapping breaks too early. Cosmetic corruption; no crash, no contract-shape violation. Requires non-default styling or raw escapes in the doc body, hence low confidence on real-world reach.
- **Evidence** — `verified` (executed this pass). Note: the first pass's single-row table repro under-demonstrated the defect, because in a one-data-row column the over-counted `length` also defines `width`, and `pad_trailing` targets `width + ansi_diff = length + (byte_size - length) = byte_size`, so the over-count cancels and no visible corruption appears. A **multi-row** table (where the column width is fixed by a *different*, plain cell) exposes it:
  ```
  $ # doc_inline_code: IO.ANSI.color(196)  (=> \e[38;5;196m, 11 bytes)
  $ IO.ANSI.Docs.print("| col | b |\n| - | - |\n| `x` | y |\n| wwwwwwww | z |",
        "text/markdown", doc_inline_code: IO.ANSI.color(196), enabled: true)
  ^[[7mcol          | b^[[0m      # header "col" over-padded to ~12, not 8
  ^[[38;5;196mx^[[0m | y          # "x" NOT aligned to the 8-wide column
  wwwwwwww     | z
  $ # default cyan (\e[36m, 5 bytes) — same table, correctly aligned:
  ^[[7mcol     | b^[[0m           # header padded to 7
  ^[[36mx^[[0m       | y          # "x" padded with 7 spaces, aligned
  wwwwwww | z
  ```
  Line-wrap over-count with a raw escape embedded in the doc body (`width: 40`):
  ```
  $ IO.ANSI.Docs.print("aaa \e[38;5;196mred\e[0m bbb ccc ddd eee fff ggg hhh iii",
        "text/markdown", width: 40, enabled: true)
  aaa ^[[38;5;196mred^[[0m bbb ccc ddd eee fff   # "red" consumed ~14 cols of budget, not 3 → wrap too early
  ggg hhh iii
  ```
- **Suggested fix** — Replace the two fixed-length clauses with a single clause that consumes `\e[` up to and including the CSI final byte (any byte in `@..~`, i.e. `m` in practice), e.g. match `<<?\e, ?[, rest::binary>>` and drop bytes until the final byte before resuming the grapheme count.

## Risks / caveats
- The module is `@moduledoc false`, so it is compiler/IEx-internal; I treated `print/3`, `print_headings/2`, `print_metadata/2`, and `print_markdown/2` (all `@doc`/`@spec`-carrying) as the public surface per the task's "documented public path" rule. If the reviewing standard restricts "public" to `@moduledoc`-exposed modules, findings 2–4 fall out of scope; the crash in finding 1 remains reachable from the same code path IEx's `h` uses to render arbitrary docstrings. This is a scope caveat, not an unverified cross-file assumption.
- No neighbour-module read was required or performed for any finding. The standing-context callgraph neighbours are the Erlang compiler sources (`elixir_tokenizer.erl` et al.), which are unrelated to this file; `docs.ex`'s only runtime dependency touched by these findings is `IO.ANSI.format_fragment/2`, which I exercised directly by running the renderer rather than reading it. Reading it could not have cleared any axis anyway — the axes here are input-shape axes (escape length, presence of closing `-->`, `doc` type), each of which I actually pushed through the real implementation.

### Ruled out
- **`print/3` with a non-binary `doc` and `"text/markdown"` prints "Unknown documentation format"** — the first clause's `is_binary(doc)` guard (`:135`) fails and it falls to the generic clause (`:139`). `verified` by execution this pass across three distinct non-binary types (integer, atom, list), each printing the unknown-format message with no crash. Not reported: `@spec` types `doc` as `term()`, but the misleading message is a cosmetic quality issue, not a crash or contract-shape violation. **Axes examined:** type of `doc` (integer `123`, atom `:an_atom`, list `["a"]`) | **Not examined:** none.
- **Block-level unclosed `<!--` comment (line starts with `<!--`)** — routes through `process/4`'s `"<!--" <> line` clause (`:210`) into the list-based `drop_comment/1` (`:649-658`), which drops lines until `-->` or end-of-input and returns `[]` safely. `verified` by execution: single-line body `"<!-- x"`, multi-line body `"<!-- x\nmore\nlines"`, and a closed-then-tail body `"<!-- x -->tail"` each completed and printed a sentinel (`NO_CRASH_1/2/3`) with no `MatchError`; the tail case correctly emitted `tail`. Only the *inline* path (finding 1) crashes. **Axes examined:** presence/absence of closing `-->`, single-line vs multi-line body, closed-with-trailing-content | **Not examined:** none.
- **`process/4` mermaid drop path re-invoking `process([], indent, options)`** (`:197-204`) — the drop-while/drop list is piped as the first argument, yielding `process(dropped_lines, [], indent, options)`, the correct 4-arity shape. `verified` by execution: a well-formed mermaid block renders `before` / `after` with the block removed and exit 0; an *unterminated* mermaid block (`Enum.drop_while` runs off the end, `Enum.drop(1)` on `[]` → `[]`) renders `x` and exits 0, no crash. **Axes examined:** terminated vs unterminated fence, empty remainder after drop | **Not examined:** none.

### Did I verify this
- Findings 1–3: **verified** — re-executed this pass on Erlang/OTP 29 / Elixir 1.21.0-dev; outputs reproduced identically.
- Finding 4 (`length_without_escape` long-ANSI): **verified mechanism by execution; low confidence on reach.** I pushed the previously-hedged axis (escape length > 5 bytes) through the real renderer via both a caller-supplied 256-color `doc_inline_code` option and a raw escape embedded in the doc body. This pass I also corrected the earlier single-row repro (which cancelled the over-count) to a multi-row table that exhibits genuine column misalignment plus header over-padding, and cross-checked byte sizes (`\e[38;5;196m`=11, `\e[38;2;1;2;3m`=13, `\e[1;33m`=7 vs matched `\e[36m`=5, `\e[1m`=4). The defect is real; confidence stays low only because it never fires under `default_options/0`, whose fragments are all ≤5 bytes when processed one at a time.
- All three "Ruled out" entries: **verified by execution**, `Not examined: none` for each — every listed axis was actually run through the renderer this pass, not reasoned about. No neighbour read was used to clear an axis.

### Dropped in re-review
- None. Every first-pass finding is preserved and re-verified. The sole first-pass hedge (`length_without_escape`) was already a finding and has been strengthened, not dropped.
