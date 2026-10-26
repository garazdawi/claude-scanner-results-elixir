# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code/fragment.ex
<!-- sha256: 781e647f549b894eaf75d2e65174a1ebfb59ed9caff41b4b55e8ea0a4e1006f5 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code/fragment.ex
<!-- sha256: 781e647f549b894eaf75d2e65174a1ebfb59ed9caff41b4b55e8ea0a4e1006f5 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

# Security Review — `Code.Fragment` (`lib/elixir/lib/code/fragment.ex`)

## Summary
Two verified algorithmic denial-of-service issues are reachable through the documented public API on untrusted source fragments (input-editor/language-server tooling feeds these functions): a quadratic blow-up in alias-chain handling and unbounded, non-optional atom interning in `container_cursor_to_quoted/2`. No code-execution, injection, or crash bugs were found. In this re-review I resolved the first-pass cross-file hedges by reading the actual delegate module (`unicode/tokenizer.ex`, i.e. `String.Tokenizer`): the identifier-security dismissal is now verified against that module's real return contract rather than assumed. Both DoS findings were already execution-verified and are preserved unchanged.

## Security

#### `CSCAN-fragment-nested-alias-quadratic`
- **Finding ID:** `CSCAN-fragment-nested-alias-quadratic`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `fragment.ex:504-521` (`nested_alias/3`), quadratic `++` at lines 509, 512, 518, 521; reached via `cursor_context/2` and `surround_context/3`.
- **Class:** Resource consumption (algorithmic-complexity DoS, CWE-407/1333-adjacent)
- **Issue:** `nested_alias/3` builds a dotted alias by re-concatenating the growing accumulator on the left of `++`: `prev ++ ~c"." ++ acc` (confirmed at 509/512/518/521 in this re-read). Because `prev` grows by one segment per recursion level and `++` copies its entire left operand each time, an N-segment alias chain costs `1+2+…+N = O(N²)`. N is bounded only by input length. The `dot`/field path (line 534 onward) builds nested tuples instead of concatenating and stays linear — the defect is specific to the alias/struct-alias branches.
- **Impact / threat:** A malicious source file containing a long alias chain (`A.A.A…`) hangs any editor tooling that calls `cursor_context/2` (autocompletion) or `surround_context/3` ("mouse-over and highlight", per the docstring at line 626) near that position. The `last_line`/`prepend_cursor_lines` reconstruction joins incomplete lines, so the payload need not be one physical line. Trust boundary: untrusted file → language-server/editor process that analyzes it. ~64 KB of input already costs ~3 s; ~256 KB costs ~1 min wall-clock hang per request.
- **Evidence:** `verified` (executed timings, first pass, unchanged).
  - `cursor_context("A.A.…A")`: `n=4000 → 37 ms`, `8000 → 183 ms`, `16000 → 683 ms`, `32000 → 2901 ms` (≈4× per 2× input = O(n²)).
  - `surround_context(s,{1,col})`: `4000 → 34 ms … 32000 → 3385 ms` (quadratic).
  - Multi-line `"A.\n"×n` (reconstructed): `4000 → 35 ms … 32000 → 3176 ms` (quadratic).
  - Control `a.a.a…` (dot path): `8000 → 1.2 ms … 64000 → 8.9 ms` (linear).
  - Struct branch `%A.A.…` (line 509): `8000 → 172 ms … 32000 → 2802 ms` (quadratic).
- **Suggested fix:** Accumulate alias segments as a list-of-charlists (iolist) and flatten once when the context is finalized, so the growing side is never repeatedly copied. Apply to all four `prev ++ ~c"." ++ acc` sites.

#### `CSCAN-fragment-container-atom-exhaustion`
- **Finding ID:** `CSCAN-fragment-container-atom-exhaustion`
- **Severity:** Low
- **Confidence:** high
- **Location:** `fragment.ex:1278` (`Keyword.take(opts, [:columns, :token_metadata, :literal_encoder])`) → `:elixir_tokenizer.tokenize` / `:elixir.tokens_to_quoted` at 1285/1300/1329 in `container_cursor_to_quoted/2`.
- **Class:** Atom creation (CWE-770, atom-table exhaustion)
- **Issue:** `container_cursor_to_quoted/2` interns one atom per unique identifier in the fragment (inherent to producing quoted AST) and provides **no** opt-out: line 1278 restricts caller options to `[:columns, :token_metadata, :literal_encoder]`, so `:static_atoms_encoder` / `:existing_atoms_only` — the atom-safety controls that `Code.string_to_quoted/2` honors via the same tokenizer path — are silently discarded before reaching the tokenizer. Atoms are never garbage-collected; exhausting the atom table raises an uncatchable `system_limit` that takes down the whole BEAM node.
- **Impact / threat:** A language server calling this on untrusted source has no atom-safe path. A file with many unique identifiers (or an editing session re-parsing accumulating identifiers) grows the global atom table monotonically toward the ~1M default limit and then crashes the tooling VM. Trust boundary: untrusted file → editor/LSP process. Weaker than the quadratic finding: atom interning is the documented behavior of any Elixir parse, so the file-specific defect is the *missing mitigation*; a practical node crash needs large (~10 MB+) or sustained input.
- **Evidence:** `verified` for atom growth and option-stripping (executed, first pass); node-crash consequence is an asserted extrapolation from measured growth (not driven to a real crash — that would kill the sandbox).
  - `container_cursor_to_quoted(frag)` with 50 000 unique identifiers: `atom delta = 50286`.
  - `Code.string_to_quoted(frag, static_atoms_encoder: enc)` → `delta 98`; `container_cursor_to_quoted(frag, static_atoms_encoder: enc)` → `delta 20001`; `container_cursor_to_quoted(frag, existing_atoms_only: true)` → `delta 20001` (both options ignored — confirms the `Keyword.take` at line 1278 strips them; the stripping is now also confirmed by direct re-read of that line).
- **Suggested fix:** Add `:static_atoms_encoder` and `:existing_atoms_only` to the allow-list at line 1278 so callers parsing untrusted fragments can suppress atom interning, mirroring `Code.string_to_quoted/2`. (Fix is a suggestion, not applied/re-tested; the underlying claim that the options are dropped is execution-verified.)

### Ruled out

- **Bidi / confusable / mixed-script identifiers (cursor_context / surround_context path)** — Basis `verified` (upgraded from a first-pass cross-file assumption). The first pass asserted "sanitation is owned by `:elixir_tokenizer`" without reading the delegate. Corrected: the cursor/surround identifier path delegates to **`String.Tokenizer.tokenize/1`** (`fragment.ex:460, 483`), which lives in `unicode/tokenizer.ex`, *not* `:elixir_tokenizer`. I read that module: `tokenize/1` returns either a 6-tuple `{kind, acc, rest, length, ascii?, special}` or an error — `{:error, {:mixed_script, …}}` (tokenizer.ex:521), `{:error, {:unexpected_token, …}}` (tokenizer.ex:445/485/530), or `{:error, :empty}` (tokenizer.ex:410). In `tokenize_identifier` (`fragment.ex:483-501`) the `case` matches only 6-tuples; **every `{:error, …}` variant falls through to `_ -> :none`**, so a mixed-script or non-conforming identifier yields no suggestion and no crash. Non-ASCII aliases are additionally rejected at `fragment.ex:489` via the `ascii?=false` flag (`{:alias, _, _, _, false, _} -> :none`), matching the ALIAS_ASCII_ONLY obligation. Load-bearing reason the class is out of scope: `cursor_context/2` and `surround_context/3` return charlist **suggestions** to the API caller as data — this file makes no eval/compile/atom decision on that text, so there is no Trojan-Source/confusable *sink* here. The `container_cursor_to_quoted/2` path, which does reach the real `:elixir_tokenizer`, is covered by the crash-fuzz entry below (bidi U+202E, NEL, LS, Hebrew all exercised, zero exceptions).

- **`lines/1` quadratic binary append** — Basis `verified`. Single-line inputs 100 K→1.6 M bytes scale linearly (0.43→6.45 ms); the ERTS binary-append optimization applies to the `<<acc::binary, char>>` accumulator. Axes examined: input length. Not examined: none — the concern is purely the append-copy cost, which the linear scaling directly refutes.

- **Crashes / unhandled inputs across all four public functions** — Basis `verified`. 300 K random-byte iterations plus 300 K token-shaped iterations (bidi U+202E, NEL, LS, Hebrew, CRLF, unbalanced delimiters, `preserve_sigils`, `trailing_fragment` combinations) over `cursor_context/2`, `surround_context/3`, `container_cursor_to_quoted/2`, `lines/1` → zero exceptions/throws. `hd/1` uses are all in guards (fail-to-next-clause) or `[] or hd(...)`-short-circuited bodies (confirmed at 577, 584, 591, 607).

- **`container_cursor_to_quoted/2` structural blow-up** — Basis `verified`. Nested open-parens and nested `do` blocks (with `trailing_fragment`) scale linearly to 32 K/16 K nestings; the token post-processing (`drop_tokens`, `reverse_tokens`, `Enum.split_while/drop_while` at 1290/1307/1321) is linear.

- **`dot`-chain quadratic** — Basis `verified`. `a.a.a…` builds nested tuples (`dot/3`, 534-568), not concatenations; linear to 64 K segments (executed control above).

- **Code execution / file / path / shell / atom-via-`String.to_atom`** — Basis `verified` (in-file). Grep for `String.to_atom`/`List.to_atom`/`Tokenizer` (this re-review) shows the only atom-creating/tokenizing calls are the `String.Tokenizer.tokenize` and `:elixir_tokenizer.tokenize`/`tokens_to_quoted` delegations already accounted for; no `String.to_atom`, no file/path/shell sink. The returned AST is documented "not safe to evaluate," which is the caller's contract, not a sink in this file.

## Risks / caveats
- Both findings are **availability** (DoS) issues against editor/language-server tooling, not confidentiality/integrity; there is no code-execution or injection surface in this file.
- Severity assumes tooling that runs these functions on files from untrusted repositories. If a caller only ever runs them on the developer's own trusted buffer, both drop to informational.
- The node-crash consequence of the atom finding is reasoned from measured per-identifier atom growth, not driven to an actual table-exhaustion crash; the growth rate itself is verified.
- Threat-model scope note (not a code hedge): whether opening a malicious file *auto-triggers* the quadratic path depends on the consuming LSP/editor's semantic-token/hover cadence. Those consumers are not in this repository's callgraph, so this cannot be verified here by reading a neighbour; within this file the APIs are documented for exactly this interactive, per-fragment use, and the DoS itself is execution-verified independent of trigger cadence.

## Did I verify this
- **`nested_alias` quadratic** — Verified: re-read lines 504-521 (four `prev ++ ~c"." ++ acc` sites confirmed); execution timings show O(n²) for alias/struct branches and linear for the dot control.
- **Atom exhaustion / option-stripping** — Verified: re-read `container_cursor_to_quoted/2` (1275-1336); line 1278 `Keyword.take` confirmed to exclude `:static_atoms_encoder`/`:existing_atoms_only`; executed deltas confirm the options are ignored while `Code.string_to_quoted` honors them.
- **Bidi/mixed-script dismissal** — Verified by reading the actual delegate `unicode/tokenizer.ex` (`String.Tokenizer.tokenize/1`) and confirming its error-return variants fall through to `:none` in `fragment.ex:483-501`; corrected the first-pass misattribution to `:elixir_tokenizer`. No axis-narrowing was performed by reading (none was needed — this is a qualitative "return is display data, no sink" dismissal, not an equivalence entry with untried axes).
- **Crash/structural/lines dismissals** — Verified by prior executed fuzzing/scaling runs; no cross-file assumption underlies them.

## Upgraded from hedged assumption in first pass
- The **Bidi / confusable / mixed-script** ruled-out entry previously rested on the unverified cross-file assumption that `:elixir_tokenizer` owns byte sanitation for this path. I read the real delegate (`String.Tokenizer` in `unicode/tokenizer.ex`), corrected the module attribution, and confirmed the dismissal against its actual return contract. The entry moves from `asserted` to `verified` and remains a dismissal (no sink), so no new finding is raised.

## Dropped in re-review
- None. Both prior findings are preserved; no prior finding was dropped.
