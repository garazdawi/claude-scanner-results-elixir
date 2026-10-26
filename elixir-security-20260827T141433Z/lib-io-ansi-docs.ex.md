# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/io/ansi/docs.ex
<!-- sha256: ad91a77794e7ba8e9b386a82f0e2ec72f339e22324d4980c36581da6a4fbffc0 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:22:23Z_

All findings verified by execution. I have what I need to write the report.

Root-cause note for the regex: bounding the scheme run (`{0,31}`) drops it from 8684ms→4.9ms while still escaping real links; a possessive quantifier does *not* help (the cost is O(n) start positions × O(n) run, not intra-match backtracking).

---

# Security Review: `lib/elixir/lib/io/ansi/docs.ex`

## Summary
`IO.ANSI.Docs` (the terminal Markdown renderer used by IEx `h` and `mix help`) has no code-exec/atom/file sinks, but its documented `print/*` entry points crash and blow up quadratically on hostile documentation text, and pass raw terminal control bytes straight through.

## Security

The trust boundary throughout: the `doc`/`headings` string is documentation content. When a developer views a **third-party dependency's** docs (`h Dep.func`, `mix help sometask`), that content is authored by the dependency, not the developer — it reaches `print/3` / `print_markdown/2` / `print_headings/2` verbatim.

#### `CSCAN-docs-inline-comment-matcherror`
- **Finding ID:** `CSCAN-docs-inline-comment-matcherror`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `docs.ex:644-645` (`drop_comment/1`, binary clause), reached from `docs.ex:695-696` (`handle_inline/5`)
- **Class:** Resource consumption (unhandled-input crash / DoS)
- **Issue:** The inline path `handle_inline("<!--" <> rest, nil, ...)` calls the *binary* `drop_comment/1`, which does a strict two-element match `[_comment, rest] = :binary.split(line, "-->")`. When a paragraph contains an opening `<!--` **not at the start of a line** and with no `-->` after it, `:binary.split/2` returns a single-element list and the match raises `MatchError`. (The line-*start* path uses the safe multi-line `drop_comment/1` list clause, so only the inline path is affected.)
- **Impact / threat:** Any documentation paragraph with a stray, mid-line, unterminated `<!--` (adversarial *or* an author's honest mistake) aborts the render. IEx's evaluator rescues it (the doc simply fails to display), but any non-IEx caller of this renderer (a Mix task, a script) is aborted. Crossing the dependency-docs → developer-terminal boundary.
- **Evidence:** `verified`.
  - `IO.ANSI.Docs.print("foo <!-- bar baz", "text/markdown", enabled: false)` →
    `** (MatchError) no match of right hand side value: [" bar baz"]` at `docs.ex:645: drop_comment/1` ← `docs.ex:696: handle_inline/5`.
  - Same crash with default options: `print("see <!-- todo notes here", "text/markdown", [])` → `MatchError`.
  - Control: `"foo <!-- bar --> baz"` and line-start `"<!-- bar baz"` → no crash.
- **Suggested fix:** Make the binary clause total, e.g. `case :binary.split(line, "-->") do [_, rest] -> rest; [_] -> "" end` (treat an unterminated inline comment as running to end-of-text).

#### `CSCAN-docs-table-cell-quadratic`
- **Finding ID:** `CSCAN-docs-table-cell-quadratic`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `docs.ex:427-443` (`table_lines/2`, `col_widths`), `docs.ex:529-546` (`generate_table_cell/1`)
- **Class:** Resource consumption (quadratic output amplification / memory DoS)
- **Issue:** `col_widths` is the max *visible* width of each column across all rows, and every cell in that column is `String.pad_leading/trailing`-padded to that width — including empty pad cells added by `pad_to_number_of_columns`. A table with `N` rows and one row containing a width-`W` column produces `O(N·W)` output, with no clamp to the terminal width.
- **Impact / threat:** A crafted Markdown table in a docstring expands to hundreds of MB in the caller's process (unbounded, not rescued — large enough OOMs the BEAM node). Dependency docs → developer terminal.
- **Evidence:** `verified`.
  - Doubling input quadruples output: `13.6KB → 6.4MB`, `27KB → 25.6MB` (via `print/3`, `enabled: false`).
  - A 53.7KB table (`5000` rows + one 20 000-char column) → **95.4MB output in 536ms**.
  - Root cause shown: `"AAAAAAAAAAAAAAAAAAAA | b\n--- | ---\nx | y"` renders the `x` cell as `"x" <> 19 spaces` — padded to the wide column's width.
- **Suggested fix:** Clamp each entry of `col_widths` to the terminal width (`min(width, options[:width])`) — a column wider than the terminal is never useful, and this bounds output to `O(N · width)`.

#### `CSCAN-docs-link-regex-redos`
- **Finding ID:** `CSCAN-docs-link-regex-redos`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `docs.ex:635-638` (`escape_underlines_in_link/1`), regex `~r{[a-z][a-z0-9\+\-\.]*://\S*}i`
- **Class:** Resource consumption (quadratic regex / ReDoS)
- **Issue:** The unanchored pattern greedily consumes a long `[a-z0-9+.\-]*` run then requires the rare suffix `://`. On a long scheme-like run that never reaches `://` (e.g. `aa…a:/`), the engine re-scans the run from every start offset — `O(n)` starts × `O(n)` run = quadratic. Applied to whole joined paragraphs and to every table column, so a single long line reaches it.
- **Impact / threat:** A long non-space run in a docstring hangs the rendering process (blocks its scheduler). Not rescued as a timeout. Dependency docs → developer terminal.
- **Evidence:** `verified`, through the public API.
  - `print(String.duplicate("a", k) <> ":/", "text/markdown", …)`: `20K→139ms, 40K→545ms, 80K→2183ms, 160K→8684ms` — perfect `4×`-per-doubling quadratic (≈85s at 500KB).
  - Fix validated: bounding the scheme run to `{0,31}` drops 160KB from `8684ms → 4.9ms` and still escapes real links (`http://a.com/x_y_z` → `x\_y\_z`). A possessive quantifier does **not** help (still ~10s), confirming the cost is start-position re-scan, not intra-match backtracking.
- **Suggested fix:** Bound the scheme-character run, e.g. `[a-z][a-z0-9\+\-\.]{0,31}://\S*` (schemes are short); optionally also cap the per-line length before link processing.

#### `CSCAN-docs-terminal-escape-passthrough`
- **Finding ID:** `CSCAN-docs-terminal-escape-passthrough`
- **Severity:** Low
- **Confidence:** high
- **Location:** `docs.ex:768-774` (`handle_inline/5` catch-all) and all `IO.puts`/`write` sinks; no input sanitisation anywhere in `print_markdown/2` / `print_headings/2`
- **Class:** Template / interpolation (terminal control-character injection, CWE-150)
- **Issue:** Raw C0/escape bytes in the doc (ESC `0x1B`, BEL `0x07`, OSC/CSI sequences) are copied through to stdout untouched. The module strips *its own* `\e[…m` codes for width math (`length_without_escape/2`) but never neutralises *input* control bytes — and does so even when `enabled: false` is set to disable ANSI.
- **Impact / threat:** A dependency's docs can emit arbitrary terminal escapes when a developer runs `h Dep.func` / `mix help`: set the window title, clear the screen, or forge/hide output so rendered docs impersonate trusted `mix`/IEx text. This is a spoofing/integrity vector distinct from the compile-time code a dependency already runs.
- **Evidence:** `verified`. `print("Hello \e]0;PWNED-TITLE\a\e[2Jworld …", "text/markdown", enabled: false)` → output bytes contain `27` (ESC) and `7` (BEL); `String.contains?(out, "\e") == true` even with color disabled. `print_headings(["title \e]0;X\a inject"], enabled: false)` likewise passes ESC through.
- **Suggested fix:** Strip/replace C0 control characters other than `\t`/`\n` (and lone ESC) from input text at the `print_markdown/2` / `print_headings/2` boundary before writing.

### Ruled out
- **`remove_square_brackets_in_link/1`** `~r{\[([^\]]*?)\]\((.*?)\)}` — *not* ReDoS. `verified`: `80KB` of `[[[[…`, and `[x]()()()…` inputs all `0.0ms`; the two lazy groups aren't nested and `.` is line-bounded.
- **`handle_inline/5` emphasis scan** — linear. `verified`: `200 000` `*` chars → `12ms` (50K→4ms, 100K→5.6ms).
- **`split_into_columns` `~r{(?<!\\)\|}`** — the table blowup is padding, not this split; the regex itself is linear.
- **`print_headings/2` `String.pad_leading(heading, div(width + len, 2))`** — for an over-length heading the target padding is `< len`, so `pad_leading` is a no-op; no amplification.
- **Atom creation / `Code.eval*` / `apply` / file & command execution / deserialisation** — none present; this module is pure text-to-ANSI formatting (grep of the file confirms no `to_atom`, `Code.`, `System.`, `File.`, `:erlang.binary_to_term`, `apply`).

## Risks / caveats
- **Trust-boundary weighting.** The realistic attacker is a third-party dependency whose docstrings a developer views. Such a dependency already executes arbitrary code at *compile* time, which caps the marginal severity of the two DoS findings (a malicious dep has stronger options) — hence Medium, not High. They remain worth fixing because (a) `CSCAN-docs-inline-comment-matcherror` also fires on honestly-malformed docs, and (b) all three are reachable by *any* code that feeds untrusted Markdown to this general renderer. The terminal-injection finding is a genuinely distinct integrity/spoofing vector not subsumed by compile-time execution.
- **`@moduledoc false`.** The module is compiler-internal; I treated the `@doc`-annotated `print*`/`default_options` functions (called by IEx/Mix) as the public surface. If the intended contract is strictly "trusted input only," the DoS findings degrade to robustness bugs — but the input demonstrably originates from third parties.
- **Not exhaustively fuzzed.** I targeted the sinks the inventory flags (regex, padding, control chars, crash clauses) and reproduced each. I did not fuzz every `handle_inline` state-machine transition; other malformed-input crashes may exist beyond the confirmed `drop_comment/1` one. All severity-bearing claims here are `verified` by execution against the build under review (Elixir 1.21.0-dev / OTP 29).

