# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code/normalizer.ex
<!-- sha256: ef557f64d1176edd0cafe39a34561d1928543c7164a516e37c2cab989373cbf5 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:58:03Z_

# Bug-hunt review — `Code.Normalizer` (`lib/elixir/lib/code/normalizer.ex`)

## Summary
The module is robust across the literal-escape and parsed-AST paths I exercised; the one genuine defect is a type-inconsistent line-metadata fallback in `normalize_kw_args/3` that emits a structurally invalid `:line` value.

## Bugfixes

#### `CSCAN-normalizer-kw-line-meta-list`
- **Finding ID** — `CSCAN-normalizer-kw-line-meta-list`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `normalizer.ex:524-525`, `normalize_kw_args/3` (first clause, the `key_meta[:format] == :keyword and not keyword?` branch)
- **Issue** — `line = key_meta[:line] || meta_line(state)` mixes two incompatible types. `key_meta[:line]` is an **integer**, but the fallback `meta_line/1` returns a **keyword list** (`[line: n]` or `[]`, per lines 610-616). The result is then used as the *value* of `:line` in `{:__block__, [line: line], …}`. When the wrapped key has no `:line` in its meta, the produced node gets `[line: [line: 5]]` (or `[line: []]`) — an AST meta whose `:line` is a list, not an integer, violating the meta contract every other site in this file upholds (e.g. line 501 `[line: meta[:line]]`, line 551 `meta_line(state)` used *as* the meta).
- **Class** — Unclassified (type-inconsistent / malformed metadata)
- **Impact** — A caller of the documented public API (`Code.quoted_to_algebra/2` → `normalize/2`, `Macro.to_string/2`) that passes AST already carrying `{:__block__, [format: :keyword], [key]}` key wrappers *without* a `:line` in that meta, inside a list that is not a full keyword list, gets an emitted node with a nested-list `:line`. Downstream the formatter treats that `:line` as a line number in comparisons and map-keys (`formatter.ex:2427-2456`, `line/1` at `2493`): since a list sorts above any integer, comment-attachment and keyword-forcing heuristics silently make the wrong grouping decision. It does not crash (the affected code paths compare/hash rather than do arithmetic on it), so the cost is subtly wrong rendering, not a failure.
- **Evidence** — `verified`. With `crafted = [{{:__block__, [format: :keyword], [:a]}, 1}, {{:__block__, [], [:b]}, 2}]`, `Code.Normalizer.normalize(crafted, line: 5)` produced `… {:__block__, [line: [line: 5]], [{{:__block__, [], [:a]}, 1}]} …` (nested-list line). Adding `line: 9` to the key meta instead produced the correct `{:__block__, [line: 9], …}`. Commands and outputs shown above (`t4.exs`, `t7.exs`).
- **Suggested fix** — Make both operands integers, e.g. `line = key_meta[:line] || state.parent_meta[:line]`, or build the meta with the helper directly: `{:__block__, meta_line(state), [{{:__block__, key_meta, [key]}, value}]}` (mirroring line 551) so `:line` is always an integer or absent.

## Risks / caveats
- The confirmed finding is reachable only when the input AST already contains `format: :keyword`-tagged key wrappers lacking a `:line` inside a non-keyword list — natural for callers re-processing formatter-shaped AST, unusual for freshly `quote`d input, hence Low. I verified the malformed metadata is produced; the *downstream* mis-rendering is reasoned from the formatter's use of that field, not reproduced end-to-end with a comment-bearing input, so the impact half is `asserted`.
- I exercised the two principal public paths — the `escape: true` literal path (maps, nested/large structures, tuples of arity 0/2/3/4, charlists, floats, quoted atoms/strings) and the parsed-AST path (defs, `case`/`receive`/`with`/`try`/`for`/`fn`, pipes, sigils, string/charlist/module-attribute interpolation) — via `Code.Normalizer.normalize |> Macro.to_string`; none crashed or mis-rendered.
- The installed toolchain (`/opt/elixir`, 1.21.0-dev) is byte-identical to the file under review (`diff` → IDENTICAL), so all reproductions run the exact code reviewed.

### Ruled out
- **Tuple keyword spread (`do_normalize({:{}, …})`, lines 196-206), `args ++ kw_list` flattening the keyword pairs into tuple positions** — intended: `{1,2,a: 1,b: 2}` is the valid formatter shape for a 3-element tuple whose last element is a keyword list; verified `{:{}, [], [1,2,[a: 1,b: 2]]}` renders `{1, 2, a: 1, b: 2}`. Basis: `verified`.
- **`CaseClauseError` in the alias branch of `normalize_literal/3` (lines 263-271) when `classify_atom == :alias` but the string isn't `"Elixir"`-prefixed** — `Macro.classify_atom` returns `:alias` only for `Elixir`/`Elixir.`-prefixed atoms; tested `:"Elixir"`, `Foo`, `Foo.Bar`, `:"Foo Bar"` (→ `:quoted`), none escape the two case clauses. Basis: `verified`.
- **Range clause (lines 58-69) mishandling default/decreasing steps** — `10..1//-1`, `1..10`, `1..10//2` all round-trip correctly; `step == 1` collapse to `..` is semantics-preserving. Basis: `verified`.
- **`meta` variable leaking out of the `if` in `normalize_kw_args/3` (line 538 vs. 551)** — Elixir does not leak `if`-branch bindings; line 551 re-reads `meta` from the `with`'s own `{:__block__, meta, _}` match, and line 538's `meta` is scoped to the taken branch. Basis: `asserted` (language scoping rule) plus the passing renders above.
- **Empty-args `normalize_call/2` (`List.last([])`, `List.pop_at([], -1, [])`)** — nil/`[]` defaults flow through the `cond`/`case` to `{form, meta, []}` without error; exercised via zero-arity calls in the parsed-AST batch. Basis: `verified`.

