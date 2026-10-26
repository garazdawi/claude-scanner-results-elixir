# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code/formatter.ex
<!-- sha256: b222f82dced7a0e29a9ec451749bc4041772a88999a3c3277ecd1ec6dd993e4e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:21:04Z_

I have verified everything I need. The huge-integer case (188 ms for 800K digits) is only mildly superlinear and dominated by bignum parsing in the tokenizer, not this file — not a meaningful finding. Let me write the report.

---

# Security review — `lib/elixir/lib/code/formatter.ex`

## 1. Summary
`Code.Formatter` (the pretty-printer behind the documented `Code.format_string!/2`, `Code.format_file!/2`, and `mix format`) contains two verified algorithmic-complexity (quadratic) DoS vectors reachable from untrusted source text; no code-execution, injection, unsafe-atom, or path sinks were found.

## 2. Security

#### `CSCAN-formatter-comment-hash-run-quadratic`
- **Finding ID**: `CSCAN-formatter-comment-hash-run-quadratic`
- **Severity**: Medium
- **Confidence**: high
- **Location**: `lib/elixir/lib/code/formatter.ex:242`, `format_comment_text/1` (the `"##" <> rest` clause)
- **Class**: Resource consumption (algorithmic complexity, CWE-1333/CWE-407)
- **Issue**: The clause `defp format_comment_text("##" <> rest), do: "#" <> format_comment_text("#" <> rest)` normalises a leading run of `#` in a comment by recursing one character at a time. Each recursion rebuilds the argument `"#" <> rest` (an O(n) binary copy) and prepends to the returned binary, so a comment consisting of a run of *k* `#` characters costs `1 + 2 + … + k = O(k²)`. Comment text is attacker-controlled and a single comment line has no length bound, so *k* can be the whole file.
- **Impact / threat**: A single comment line of `#` characters pins one CPU core for time quadratic in its length. Measured: 160 KB → 3.8 s; extrapolating at the observed 4×-per-doubling rate, ~1 MB → ~2.5 min, ~5 MB → ~1 h — from an input a few megabytes in size with no nesting or structure required. Trust boundary: untrusted Elixir source reaching `Code.format_string!/2` / `Code.format_file!/2` / `mix format` (online formatters/playgrounds, editor/LSP format-on-open of a file in an untrusted repo, `mix format --check-formatted` in CI on an untrusted PR).
- **Evidence**: `verified`.
  - Quadratic growth via the public API:
    ```
    $ elixir -e 'for n<-[40000,80000,160000], do:
        (s=String.duplicate("#",n)<>"\n"; {u,_}=:timer.tc(fn->Code.format_string!(s) end);
         IO.puts("#{n} -> #{Float.round(u/1000,1)} ms"))'
    40000  -> 76.4 ms
    80000  -> 447.0 ms
    160000 -> 3787.4 ms
    ```
  - Isolation from the tokenizer — an equal-length comment that does **not** hit the `"##"` recursion stays linear:
    ```
    n=40000:  all-# = 76.4 ms   vs   "# "+40000×'x' = 1.0 ms
    n=80000:  all-# = 447.0 ms  vs                     1.6 ms
    n=160000: all-# = 3787.4 ms vs                     4.7 ms
    ```
    The control passes through identical tokenisation, proving the quadratic term is `format_comment_text/1` in this file.
- **Suggested fix**: Consume the whole leading `#` run in a single pass instead of one byte per recursion. The existing general clause `format_comment_text("#" <> rest), do: "# " <> rest` already inserts the needed space; the `"##"` clause only needs to normalise `##…` without an O(n)-per-step rebuild — e.g. scan the maximal run of `#` once (`:binary`/counting), then build `"# " <> <run minus one '#'> <> rest` directly, or use a tail-recursive accumulator that reverses once.

#### `CSCAN-formatter-format-to-string-quadratic`
- **Finding ID**: `CSCAN-formatter-format-to-string-quadratic`
- **Severity**: Medium
- **Confidence**: high
- **Location**: `lib/elixir/lib/code/formatter.ex:2527` `format_to_string/1`; call sites `bitstring_wrap_parens/3` at `:1598` and `capture_to_algebra/3` at `:1051`
- **Class**: Resource consumption (algorithmic complexity, CWE-1333/CWE-407)
- **Issue**: `format_to_string/1` fully renders an algebra sub-document to a binary (`Inspect.Algebra.format(:infinity)`) merely to inspect a prefix/suffix — the first byte (capture: `& ` vs `&`) or the leading `~`/`<<` and trailing `>>` (bitstring paren decision). Both call sites sit on a recursive descent, so for input nested to depth *d* the entire inner document (size O(*d*)) is re-rendered at every level, giving O(*d²*). Nesting depth is attacker-controlled and effectively unbounded (the parser accepts these forms to at least depth 50 000).
- **Impact / threat**: Deeply nested single-segment bitstrings (`<< << … 1 … >> >>`) or nested captures (`& & … &1`) make formatting cost grow quadratically in nesting depth. Measured bitstrings: depth 4000 → 2.5 s, ~21× a same-depth list; such input parses at depth 50 000, extrapolating to ~6 min of CPU from a ~300 KB file. Same trust boundary and entry points as the finding above.
- **Evidence**: `verified`.
  - Bitstring quadratic and its excess over structurally-similar nesting that does *not* call `format_to_string`:
    ```
    depth=1000: bitstring=149.9ms  list=8.6ms   tuple=54.9ms
    depth=2000: bitstring=634.6ms  list=30.9ms  tuple=236.2ms
    depth=4000: bitstring=2513.7ms list=118.4ms tuple=957.9ms   (~4× per doubling; 21× the list)
    ```
  - Nested captures (same `format_to_string` mechanism):
    ```
    depth=1000 -> 34.7 ms   depth=2000 -> 126.7 ms   depth=4000 -> 470.3 ms
    ```
  - Parser accepts the depth needed to reach multi-minute cost: `Code.string_to_quoted!` on depth-20000 and depth-50000 nested bitstrings both return "parsed OK".
- **Suggested fix**: Don't fully render the sub-document to test a tiny prefix/suffix. Carry the needed shape alongside the doc during construction (e.g. return/track whether the operand's rendering begins with `~`/`<<`/a digit/`&` or ends with `>>`), or compute just the first/last character without an infinite-width full format. This removes the per-level O(*d*) re-render at both call sites.

### Ruled out
- **Atom-table exhaustion via `String.to_unsafe_atom/1` (line 1468)** — `verified` not reachable under default options: the call is inside the `{:ok, callback}` branch of `Map.fetch(state.sigils, …)`, and `state.sigils` is empty unless the caller passes a `:sigils` config entry. Formatting 5000 distinct `~SIGn` sigils under default opts created **0** formatter-attributable atoms once code paths were warmed (the `:sigil_SIGn` atoms observed come from the tokenizer, a different file). When the branch *is* taken, the name is one the caller already interned as a config key, so it is bounded by caller config, not by source.
- **Code execution via sigil callback (line 1473 `callback.(hd(entries), metadata)`)** — the callback originates from the caller's `:sigils` option, not from source text; running it is the caller's explicit choice, not an injection from untrusted input.
- **`apply(Inspect.Algebra, join, [])` (line 1271)** — `join` is an internally-computed formatting mode (`:line`/`:break`/`:flex_break`), never derived from source content; not dynamic dispatch on external data.
- **`:erlang.binary_to_atom` occurrences (lines 339, 615, 2479)** — these are AST *pattern matches* recognising the shape of an atom-interpolation node in the input in order to format it; they do not call `binary_to_atom` and create no atoms.
- **`inspect(unknown, printable_limit: :infinity)` (line 644)** — the catch-all is reached only by foreign AST terms (PIDs, refs, non-AST structures) that cannot arise from parsing source; standard nodes from `Code.string_to_quoted` are handled by earlier clauses.
- **No `Code.eval*`, `System.cmd`/`:os.cmd`, `File.*`, `:erlang.binary_to_term`, `Regex.compile`, `Module.concat`, `Path.*`, or archive sinks** — `verified` absent by search of the whole file; the only matches were the AST pattern-matches and the internal `apply` noted above.
- **`insert_underscores/do_insert_underscores` on huge integer literals** — `verified` only mildly superlinear (800 000 digits → 188 ms), dominated by bignum decimal parsing in the tokenizer, not by this file's binary accumulation; not a meaningful DoS.

## 3. Risks / caveats
- **Residual, not this file**: nested lists/tuples are themselves mildly superlinear (depth 4000 list → 118 ms) even without `format_to_string`. That baseline cost lives in `Inspect.Algebra` / general deep-recursion handling, not in `formatter.ex`; I did not chase it here. It means very deep nesting has some intrinsic cost regardless of the second finding's fix, but `format_to_string` dominates (21× at depth 4000) and is squarely this file's logic.
- **Severity framing**: both findings are single-core CPU-exhaustion DoS only — no crash, memory corruption, code execution, or state poisoning — hence Medium rather than High. The amplification (sub-MB input → minutes of CPU) and the low-trust entry points (`mix format`/LSP/online formatters on untrusted source) are what keep them above Low.
- **Extrapolations** (~1 MB → ~2.5 min for the comment case; depth-50 000 → ~6 min for the bitstring case) are computed from the measured 4×-per-doubling curve, not run to completion, to avoid multi-minute sandbox hangs; the measured points up to 160 KB / depth 4000 are direct observations and the quadratic exponent is stable across them.
- The review covered only `formatter.ex`; the tokenizer/parser it depends on (which do intern `:sigil_*` atoms and do the bignum parsing noted above) are out of scope and unexamined here.

