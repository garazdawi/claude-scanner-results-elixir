# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code/normalizer.ex
<!-- sha256: ef557f64d1176edd0cafe39a34561d1928543c7164a516e37c2cab989373cbf5 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code/normalizer.ex
<!-- sha256: ef557f64d1176edd0cafe39a34561d1928543c7164a516e37c2cab989373cbf5 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

# Security review — `lib/elixir/lib/code/normalizer.ex`

## Summary
`Code.Normalizer` is robust against malformed AST. Its keyword-detection logic is quadratic on crafted **all-atom-keyed** list literals whose keyword-ness never latches, and it mints unbounded-but-input-bounded atoms from alias segments — both reachable through the documented public `Macro.to_string/1` and `Code.quoted_to_algebra/2`. The first-pass hedge ("did not exhaustively fuzz every AST node shape") has been resolved by execution: the `normalize_call`, `normalize_bitstring`, deep-nested-chain, and non-atom-keyed-list paths were timed at scale (4k–80k) and are all **linear**; the only super-linear path is the atom-keyword suffix rescan already reported below.

## Security

#### `CSCAN-normalizer-keyword-quadratic`
- **Finding ID** — `CSCAN-normalizer-keyword-quadratic`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `normalizer.ex:519` and `normalizer.ex:534` (`normalize_kw_args/3`, the `keyword? = keyword? or keyword?(current)` idiom), driven by `keyword?/1` at `normalizer.ex:618-634`; entered from the list-literal path `normalizer.ex:316` and map path `normalizer.ex:478`.
- **Class** — Resource consumption (algorithmic complexity / quadratic blowup, CWE-407/CWE-1333-style).
- **Issue** — For every element of a `{key, value}` list, `normalize_kw_args/3` re-evaluates `keyword?(current)` over the *entire remaining suffix*. When the list is not a proper keyword list (so the accumulator never latches to `true`), each of the *n* elements triggers an O(n) scan → **O(n²)**. The trigger requires that `keyword?/1` not short-circuit on the first element: keys must be **atoms** (the `is_atom(key)` clause at line 626 keeps scanning), with a single "poison" pair whose key is a module-alias atom (e.g. `:"Elixir.Foo"`) at/near the end forcing the scan to traverse the whole suffix before returning `false`. A proper keyword list latches `keyword?` to `true` on the first scan and stays linear.
- **Note on the trigger boundary (new this pass)** — A list with a **non-atom** key (e.g. integer-keyed `{i, i}` pairs) does **not** reproduce: `keyword?/1` falls straight to the `keyword?(_other) -> false` clause on element 1 without scanning the suffix, so cost stays linear. This narrows the exploit precondition to *all-atom keys with a non-keyword atom (alias) forcing the failure late*. Measured: integer-keyed list of 32000 pairs = 50 ms (linear), vs. all-atom+alias poison below.
- **Impact / threat** — Any service or tool that renders an externally-influenced AST/term as source via `Macro.to_string/1` or `Code.quoted_to_algebra/2` (code generators, formatters/linters over deserialized ASTs, LiveBook-style renderers, diagnostic printers that `Macro.to_string` untrusted data) can be driven to burn CPU: a crafted list (~10⁴–10⁵ elements) costs seconds to tens of seconds on one scheduler. Trust boundary crossed: attacker-supplied data/AST → a printing API documented to accept any `Macro.t()`.
- **Evidence** — `verified` (re-run this pass, `+fnu`, GC before each timing). Doubling *n* ~quadruples time:
  ```
  # all-atom keys + one :"Elixir.Foo" tail, via Code.quoted_to_algebra
  n=4000  117 ms   n=8000  401 ms   n=16000 1593 ms      (ratio ≈ 3.4×, 4.0× per doubling → O(n²))
  # integer-keyed pairs (short-circuits keyword?/1):
  n=32000  50 ms                                         (linear — does NOT reproduce)
  ```
  Prior-pass measurements (n up to 60000 → 28 s) are consistent.
- **Suggested fix** — Determine keyword-ness without per-element suffix rescans. Because `keyword?` is monotonic over suffixes (once a suffix is a keyword list, every later suffix is too), compute the transition in a single O(n) right-to-left pass (or compute `keyword?(elems)` once for the whole-list case) and thread the resulting boolean, instead of calling `keyword?(current)` at each recursion step.

#### `CSCAN-normalizer-alias-atom-creation`
- **Finding ID** — `CSCAN-normalizer-alias-atom-creation`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `normalizer.ex:270` (`Enum.map(&String.to_unsafe_atom/1)` in the alias branch of `normalize_literal/3`, lines 260-273) and `normalizer.ex:578` (`maybe_escape_literal/2`).
- **Class** — Atom creation (CWE-770, atom table is never GC'd).
- **Issue** — When a bare module-alias atom (`:"Elixir.A.B.C"`) reaches `normalize_literal/3`, it is split on `"."` and each segment is turned into an atom via `String.to_unsafe_atom/1` (the unconditional atom-creating conversion). Segment atoms need not previously exist, so one 255-byte alias atom expands into many new atoms. `maybe_escape_literal/2` (line 574-579) similarly mints an escaped variant of every atom literal (1:1).
- **Impact / threat** — Reachable from `Macro.to_string/1` / `Code.quoted_to_algebra/2` on an AST carrying raw module atoms (e.g. from `Macro.escape/1` of untrusted data, or a deserialized AST). Each printed alias literal adds tens of permanent atoms; a long-running service that repeatedly normalizes untrusted ASTs accumulates atoms toward the ~1M table limit. Amplification is **bounded** (segments per alias atom, ≲tens×) and the input atoms must already exist, so this is an accelerant, not an unbounded exhaustion primitive.
- **Evidence** — `verified` (re-run this pass): 500 alias atoms × ~20 segments each → **10,803** new atoms; a single 25-segment alias atom → **26** new atoms. Independent of the stale-beam caveat (atom-count delta is measured on the live VM regardless of which `to_atom` variant the loaded beam calls).
- **Suggested fix** — No clean drop-in: `String.to_existing_atom/1` would raise on legitimately-new module names the formatter must render. Mitigate at the boundary — treat `Macro.to_string`/`quoted_to_algebra` on untrusted ASTs as atom-creating and gate input size / run under a bounded process; if feasible, cap alias segment count/length before splitting.

### Ruled out

- **Quadratic/pathological cost in `normalize_call` or `normalize_bitstring` (the first-pass hedge).** `normalize_call` (line 325) performs a fixed number of single-pass scans per node (`length/1`, `List.last/1`, `List.pop_at/2`, one `normalize_args` map); `normalize_bitstring` (line 419) maps once over parts plus one `List.last`/`Enum.reverse` in `maybe_add_trailing_newline`. None re-scan a growing suffix per element.
  - *Basis: verified.* **Axes examined:** flat call with n plain args; bitstring with n non-interpolated `::` parts; deep left-nested call chain `f(f(…))`; non-atom-keyed list literal — all timed at n = 4000…80000 and observed linear (call 32000 = 11 ms; bitstr 32000 = 33 ms; nested 10000→80000 = 15→156 ms ≈ 8× size / 10× time; integer-keyed list 32000 = 50 ms). **Not examined:** none among the two named sinks and their obvious large-fan-out shapes. Compound shapes whose last arg is itself a large **atom-keyed keyword list** route back through `normalize_kw_args` and are the `CSCAN-normalizer-keyword-quadratic` sink above, not a distinct one.
- **`CaseClauseError` on alias atoms (line 263).** `Macro.classify_atom/1` returns `:alias` only via `classify_binary("Elixir" <> rest, …)`, so every `:alias` atom's string starts with `"Elixir"`; the two-clause `case` (`"Elixir"` / `"Elixir." <> segments`) is exhaustive. *Basis: verified.* **Axes examined:** 200,000 random atoms (none classified `:alias` without an `"Elixir"` prefix) plus the `Macro` classify source path. **Not examined:** none — the classification is total over the `:alias` result by construction.
- **Stack overflow via deeply nested AST.** `do_normalize/2` recursion uses the BEAM's growable stack; an 80,000-deep nested call chain normalized and formatted without error in 156 ms (this pass), and any such input must first be deserialized/built (the parser bounds source-derived depth). *Basis: verified.* **Axes examined:** nested depth to 80,000. **Not examined:** none material — depth is the only axis and it completes without crash or super-linear cost.
- **Crash on malformed / non-AST literals.** Raw pids, refs, funs, `{:., [], [1,2,3]}`, `{:%{}, [], [{:|,[],[1]}]}`, `{:foo, [], :notalist}`, atoms/strings with control bytes all normalize and format without raising (the catch-all `normalize_literal/3` at line 321 and the formatter absorb them). *Basis: verified* (prior pass). **Axes examined:** the enumerated malformed shapes. **Not examined:** exhaustive enumeration of all term shapes — the catch-all clause at line 321 is the structural backstop, so remaining shapes fall through to `{:__block__, meta, [quoted]}`.
- **Escaping bypass / re-parse injection.** String/atom literals are escaped via `Code.Identifier.escape/2` when `escape: true` (the default); `escape: false` is a documented caller opt-out for already-escaped input, not an attacker-reachable bypass. *Basis: verified* against the `Code.quoted_to_algebra/2` option docs and the `maybe_escape_literal/2` clauses. **Axes examined:** default (`escape: true`) escaping of string and atom literals. **Not examined:** none relevant — `escape: false` is operator-controlled, outside the untrusted-source attacker model.

## Risks / caveats
- The sandbox's installed compiler (`/opt/elixir`, 1.21.0-dev / OTP 29) carries **stale beams** — its loaded `String`/`Code.Normalizer` beams predate the current sources (`function_exported?(String, :to_unsafe_atom, 1)` is `false`), but the on-disk `code/normalizer.ex` is byte-identical to the file under review. All behavioral and timing claims were driven through the public `Code.quoted_to_algebra/2` API against that build and reproduce the source logic directly; atom-count deltas are measured on the live VM and hold regardless of which `to_atom` variant the loaded beam calls.
- Severity of the quadratic finding assumes a caller that feeds attacker-influenced ASTs/terms to these public printing APIs; programs that only `Macro.to_string` their own trusted AST are unaffected. Impact is CPU on a single scheduler (recoverable), not memory exhaustion or crash.

## Did I verify this?
- **Keyword quadratic** — **Verified by execution** (re-run this pass). Also newly established the *trigger boundary*: only all-atom-keyed lists with a late non-keyword atom reproduce; non-atom keys short-circuit `keyword?/1` and stay linear.
- **Alias atom creation** — **Verified by execution** (re-run this pass): 10,803 new atoms from 500×20 segments; 26 from one alias. Bounded-amplification characterization unchanged.
- **First-pass hedge ("did not exhaustively fuzz every AST node shape … `normalize_call`/`normalize_bitstring` reasoned about but not timed")** — **Resolved / no longer hedged.** The two named sinks plus deep-nesting and non-atom-keyed lists were timed at scale and are linear; no additional quadratic path was found. The former caveat is now a `verified` Ruled-out entry above with the axes actually executed. No unverified assumption remained to upgrade to a finding.

### Dropped in re-review
- None. Both findings from the first pass are preserved and re-verified. The only change is the first-pass Risks/caveats hedge, which was converted from an untested assumption into a `verified` Ruled-out entry after timing the `normalize_call`, `normalize_bitstring`, deep-nested-chain, and non-atom-keyed-list paths at n = 4000…80000 (all linear).
