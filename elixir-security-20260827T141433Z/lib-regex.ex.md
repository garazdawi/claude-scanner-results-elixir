# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/regex.ex
<!-- sha256: b3804f759fcd8aaaae6e5031dcdde65be4299cc83a88f583a72944ce273e95b4 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/regex.ex
<!-- sha256: b3804f759fcd8aaaae6e5031dcdde65be4299cc83a88f583a72944ce273e95b4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (OTP 29 / erts-17.0.2 / Elixir 1.21.0-dev / PCRE2)_

# Security review — `lib/elixir/lib/regex.ex`

## 1. Summary
The `Regex` module is generally robust — `escape/1`, `to_embed/2`, option translation, the export/import (`E` modifier) round-trip, and the split/replace index arithmetic all held up under testing. Two caller-reachable crash paths remain: a malformed capture reference in the `replace/4` replacement string crashes with an internal `MatchError`/`ArgumentError`, and an out-of-range `:offset` passed to `run/3`/`named_captures/3` is forwarded verbatim to `:re.run`, which raises `ArgumentError`. The module also exposes no control over PCRE's match cost when patterns come from untrusted input. In this re-review I executed the previously-hedged axes: the split/named-capture index arithmetic was fuzzed (clean), the `:re.import`/export consumption path was exercised end-to-end (works), and the offset-forwarding crash was isolated to the engine. The one axis I still cannot exercise is OTP ≤ 27 / PCRE1, because only OTP 29 is installed here.

## 2. Security

#### `CSCAN-regex-replace-malformed-backref-crash`
- **Finding ID:** `CSCAN-regex-replace-malformed-backref-crash`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/elixir/lib/regex.ex:867-870` (`precompile_replacement/1`, the `\g{` clause) and `:891-898` (`pick_int/1`); reached from the documented `replace/4` at `:841-858`.
- **Class:** Resource consumption / unhandled exception (DoS on malformed replacement string)
- **Issue:** When the replacement argument to `Regex.replace/4` contains a malformed `\g{…}` group reference, the private parser crashes instead of rejecting the input cleanly:
  - `\g{}` → `List.to_integer(ns)` is called with `ns = []` → `ArgumentError`.
  - `\g{x}`, `\g{-1}`, `\g{1x}`, `\g{1` (any non-digit inside, or no closing `}`) → the tuple match `{ns, <<?}, rest::binary>>} = pick_int(rest)` at line 868 fails → `MatchError` that leaks the internal intermediate value (e.g. `{~c"1", "x}"}`).
  The documentation (lines 789-838) advertises `\g{N}` as supported replacement syntax, so this parser is on the documented public path; it just has no error branch for the malformed forms.
- **Impact / threat:** An application that calls `Regex.replace/4` with an attacker-influenced *replacement template* (user-supplied search-and-replace, a templating/notification feature, config-driven rewrite) can be crashed on demand with a one-line input. The exception is an internal `MatchError`/`ArgumentError` rather than a documented error, so it also defeats a caller's reasonable `rescue ArgumentError`. Effect is availability (crash of the calling process/request); no data exposure or code execution. Trust boundary: caller-forwarded untrusted string → library sink.
- **Evidence:** `verified` (re-run this pass, OTP 29). Driving the exact bytes through the public API:
  ```
  \g{}   -> CRASH ArgumentError: ... not a textual representation of an integer
  \g{x}  -> CRASH MatchError: no match of right hand side value: {[], "x}"}
  \g{1   -> CRASH MatchError: no match of right hand side value: {~c"1", ""}
  \g{1x} -> CRASH MatchError: no match of right hand side value: {~c"1", "x}"}
  \g{-1} -> CRASH MatchError: no match of right hand side value: {[], "-1}"}
  \g{1}  -> OK -> "b"    (well-formed form is unaffected)
  \1     -> OK -> "b"    (numeric \N form always carries ≥1 digit, so it is safe)
  ```
- **Suggested fix:** Make the `\g{` clause total. Guard `pick_int` for the digits-then-`}` shape and raise a single descriptive `ArgumentError` on empty/non-digit/unterminated content — or treat a malformed `\g{…}` as a literal (matching how a bare `\g` without braces is already handled by the catch-all clause). Concretely, replace the crashing `{ns, <<?}, rest::binary>>} = pick_int(rest)` / `List.to_integer(ns)` with `case pick_int(rest) do {[_|_] = ns, <<?}, rest2::binary>>} -> ... ; _ -> raise ArgumentError, ... end`.

#### `CSCAN-regex-offset-forwarded-argerror`
- **Finding ID:** `CSCAN-regex-offset-forwarded-argerror`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/elixir/lib/regex.ex:379,381` (`run/3` reads `:offset` and forwards it) and `:418-421` (`named_captures/3` funnels through `run/3`); the value reaches `:re.run` via `safe_run/3` at `:621`/`:623` unchecked.
- **Class:** Resource consumption / unhandled exception (DoS on out-of-range option)
- **Issue:** The documented `:offset` option (lines 352, 395) is read with `Keyword.get(options, :offset, 0)` and passed straight into `:re.run` with no bounds check. When `offset > byte_size(string)`, `:re.run` raises `ArgumentError` ("argument error"), which propagates out of `run/3`/`named_captures/3` uncaught.
- **Impact / threat:** A caller that forwards an attacker-influenced `:offset` (e.g. a pagination/scan cursor derived from a request) into `Regex.run/3` or `Regex.named_captures/3` can crash the calling process. Availability only; no data exposure. The crash is a plain `ArgumentError` raised by the engine, so it is somewhat less surprising than the `replace/4` `MatchError`, but it is still undocumented for this option and not defended in `regex.ex`.
- **Evidence:** `verified` (this pass). Surfaced by fuzzing `named_captures/3` across `offset ∈ {0,1,100}`; every `offset` past the subject length raised `ArgumentError`. Isolated to the engine:
  ```
  Regex.run(~r/a/, "abc", offset: 100)          -> CRASH ArgumentError: argument error
  :re.run("abc", "a", [{:offset, 100}])         -> CRASH ArgumentError: argument error   (bare engine, same crash)
  Regex.scan(~r/a/, "abc", offset: 100)         -> []                                     (scan/:global tolerates it)
  ```
  So the crash originates in `:re.run`, not in `regex.ex` arithmetic; `regex.ex` simply forwards the option without a guard. `scan/3` (which adds `:global`) returns `[]` for the same offset.
- **Suggested fix:** Either clamp/validate `:offset` against `byte_size(string)` before calling `safe_run`, returning `nil`/`[]` for an out-of-range offset (matching `scan/3`'s tolerance), or document that `:offset` must be `0..byte_size(string)` and that an out-of-range value raises `ArgumentError`.

#### `CSCAN-regex-untrusted-pattern-redos`
- **Finding ID:** `CSCAN-regex-untrusted-pattern-redos`
- **Severity:** Low
- **Confidence:** low
- **Location:** `lib/elixir/lib/regex.ex:219-221` (`compile/2`) → `:615-625` (`safe_run/3`, the `:re.run` calls); `run/3`, `match?/2`, `scan/3`, `split/3`, `replace/4` all funnel through here.
- **Class:** Resource consumption (ReDoS / catastrophic backtracking)
- **Issue:** `compile/2` accepts an arbitrary attacker-controlled pattern and the run-path functions accept an arbitrary subject. Elixir forwards only `:return`/`:capture`/`:offset` to `:re.run` (lines 377-381, 603-606) and never sets or exposes `:match_limit` / `:match_limit_recursion`, so a caller has no API surface to bound backtracking cost when the pattern is untrusted.
- **Impact / threat:** A pathological pattern compiled from untrusted input (e.g. `(a+)+$`) can be forced to consume an elevated amount of CPU per match — CPU-exhaustion DoS with modest request volume. Trust boundary: untrusted regex source reaching `compile/2`.
- **Evidence:** `verified on OTP 29 / PCRE2 only`, with a mitigating result: catastrophic backtracking is *not* unbounded on this build — PCRE2's default match limit aborts and returns `nil`/`:nomatch` after a roughly constant time per call:
  ```
  (a+)+$  n=40  -> nil  73ms
  (a+)+$  n=80  -> nil  75ms      (flat in n, not exponential)
  (a+)+$  n=120 -> nil  75ms
  ```
  The classic "hang forever" ReDoS is capped to a near-constant per-call cost on this build; residual risk is bounded CPU amplification, which depends on the underlying PCRE build/limits that this module neither controls nor surfaces. **Confidence is low** because the security-relevant behavior lives in the PCRE build, and the one axis that would change the verdict — the OTP ≤ 27 / PCRE1 code path that `safe_run` explicitly still supports — I could **not** exercise (only OTP 29 is installed here). On PCRE1 with no match limit set, the same pattern could backtrack far longer.
- **Suggested fix:** Largely inherent to PCRE and mostly a documentation/robustness item. Consider letting callers of `run/3`/`scan/3`/`match?/2` pass through `:match_limit`/`:match_limit_recursion` to `:re.run` (currently dropped), and/or document that patterns from untrusted sources are a DoS vector whose mitigation relies on the underlying `:re`/PCRE match limit.

### Ruled out

- **`split/3` + `named_captures/3` index arithmetic** (`do_split/6` `:739-787`, `apply_list/5` `:900-923`, `get_index/2` `:950-957`) — **Verified safe** (upgraded from the first pass's "did not fuzz" caveat). Fuzzed 8 adversarial patterns (`~r//`, `~r//u`, zero-width `(?=x)`, optional named groups, `\b`, `^`/`$` multiline, mixed named alternation) × 8 subjects (empty, ASCII, multi-byte UTF-8, newlines, 50-char) × 9 `:on` values (`:first`/`:all`/`:all_but_first`/`:none`/`:all_names`/`[:a]`/`[:x,:y]`/`[1]`/`[0,1]`) × 4 `:parts` × trim × include_captures — every combination returned without a crash. The binary-slice patterns in `do_split`/`apply_list` consume indices produced by `:re.run` (trusted producer) and stayed internally consistent across empty and zero-width matches.
  - Axes examined: pattern shape (empty/zero-width/named/anchored/unicode), subject encoding (ASCII/multi-byte/newline/empty), `:on` selector (all documented + named + numeric), `:parts`, `:trim`, `:include_captures`. | Not examined: none.

- **`escape/1` metacharacter neutralization** (`:984-1010`) — Verified safe. `@escapable` includes `.^$*+?()[]{}|#-\` **and** the whitespace bytes (`\t\n\v\f\r` and space), so escaped output matches its input literally in both normal and extended (`x`) mode; hostile inputs (`.*`, `a|b`, `(?:evil)`, `[a-z]`, `foo # comment\nbar`, `\`, `{2,3}`) each matched only themselves under `~r/…/` and `~r/…/x`.
  - Axes examined: metacharacter set, whitespace/extended-mode `#`-comment injection, backslash doubling, position 0 vs mid-string (both `escape_char` clauses). | Not examined: none.

- **`to_embed/2` group breakout** (`:512-540`) — Verified no breakout. Only a successfully-compiled (hence balanced) `source` reaches the function; it is wrapped as `(?flags-flags:source[\n])` with a trailing `\n` for extended patterns to terminate a dangling `#` comment. An unbalanced source (`a)b`) fails to compile and never reaches it.
  - Axes examined: balanced vs unbalanced source, extended-mode trailing-comment, modifier enable/disable ordering, strict vs non-strict untranslatable-option path. | Not examined: none.

- **`compile/2` option-string parsing** (`translate_options`, `:1037-1063`) — An invalid modifier returns `{:error, {:invalid_option, rest}}` cleanly; no crash, no atom creation, no injection.
  - Axes examined: each valid modifier char, invalid char, empty string, deprecated `r`, `E` presence/absence. | Not examined: none.

- **Atom creation / `String.to_atom` / `Module.concat`** — None present. `translate_options` maps only fixed characters to compile-time atoms; no external bytes are atomized.
  - Axes examined: every `translate_options` clause. | Not examined: none.

- **`:re` export/import consumption path** (`import/1` `:296-304`, `maybe_import_pattern/1` `:627-632`, `safe_run/3` else branch `:623`, `names/1` `:552-555`) — **Verified working** (upgraded from the first pass's "outside this file / treated `:re` contracts as given" hedge, for the *consumption* contract that `regex.ex` owns). Exercised the full round-trip on OTP 29: `Regex.compile!("f(o)o","E")` yields a `{:re_exported_pattern, …}`; `run/3` on it dispatches through `safe_run`'s else branch → `maybe_import_pattern` → `:re.import` and returns `["foo","o"]`; `Regex.import/1` converts the tag back to `:re_pattern`; `names/1` on the exported pattern returns `[]` (correct — no named groups). The pattern-tag guard `{:re_pattern,_,_,_,<<_::bitstring>>}` in `safe_run` correctly excludes exported patterns and routes them to the import path.
  - Axes examined: exported-vs-plain pattern dispatch in `safe_run`, `:re.import` round-trip, `import/1` opts stripping, `names/1` on an exported pattern. | Not examined: the internal PCRE representation produced by `:re.compile`/`:re.import` (engine internals, not code this file owns).

- **`__escape__/1` and `__import_pattern__/2`** (`:1065-1120`) — Both `@doc false` compile-time macros invoked when a `%Regex{}` literal is escaped into AST; reachable only through the compiler embedding a struct the builder already controls. Out of scope per the review's public-API restriction.
  - Axes examined: reachability (compile-time only), match/guard rejection in `__import_pattern__`. | Not examined: none relevant (out of public-API scope).

- **OTP-version comparison quirk in `safe_run`/`__escape__`** (`>= [?2, ?8]`, e.g. line 619; `< [?2, ?8]`, line 1074) — Charlist comparison would misorder a hypothetical OTP ≥ 100 (`[?1,?0,?0] < [?2,?8]`), but this is an internal version gate, not security-relevant, and not reachable for any current release.
  - Axes examined: current-release release strings ("27"/"28"/"29"). | Not examined: hypothetical ≥ 3-digit OTP release string (not reachable today; non-security).

## 3. Risks / caveats
- **OTP ≤ 27 / PCRE1 not exercised.** The ReDoS cap (~75 ms flat) and the offset/replace crash behaviors were all measured on OTP 29 / PCRE2 — the only toolchain installed in this sandbox. The `safe_run` shim (`:615-625`) and `__escape__` (`:1074`) explicitly still support the OTP ≤ 27 PCRE1 path, where no match limit is set; on that build the same untrusted pattern could backtrack far longer. This axis stays **not examined** — it is why `CSCAN-regex-untrusted-pattern-redos` is rated low confidence rather than ruled out.
- **`:re` engine internals not re-derived.** I verified how `regex.ex` *drives and consumes* `:re.run`/`:re.compile`/`:re.import` (export/import round-trip, offset forwarding, match-limit behavior) by executing them, but the internal PCRE matching/compilation implementation is not code this folder owns and was treated as a trusted external engine.
- **Replacement-crash severity is context-dependent.** `CSCAN-regex-replace-malformed-backref-crash` is Low because it is an isolated supervised-process crash with no data/code impact; in a deployment where `Regex.replace/4` templates come directly from end users on a hot path, availability impact could be higher.

## 4. Did I verify this?
- **Replace malformed-backref crash** — Verified by execution (drove all five malformed `\g{…}` forms plus the two well-formed forms through `Regex.replace/4`). Confidence high.
- **Offset out-of-range crash** — Verified by execution; isolated to `:re.run` (bare engine reproduces it) and confirmed forwarded unchecked from `run/3`/`named_captures/3`. Confidence high.
- **Split / named-capture index arithmetic** — Verified by ~9,000-combination fuzz (patterns × subjects × on × parts × trim × include); no crash. Confidence high that the in-file arithmetic is total over well-formed `:re.run` output.
- **Export/import consumption path** — Verified by executing the `E`-modifier round-trip end-to-end on OTP 29. Confidence high.
- **ReDoS bound** — Verified on OTP 29 / PCRE2 only; OTP ≤ 27 not testable here. Confidence low overall (build-dependent).
- **escape / to_embed / translate_options** — Verified by execution and by exhausting the clause axes. Confidence high.

## Dropped in re-review
- None. All findings from the first pass are preserved. The first pass's two hedged caveats were resolved rather than dropped: the "did not fuzz split/named_captures" caveat was discharged by executing the fuzz (→ moved to Ruled out as verified), and the "treated `:re` contracts as given / outside this file" caveat was discharged for the consumption contract by executing the export/import round-trip (→ Ruled out as verified), leaving only the genuinely un-exercisable engine-internals and OTP ≤ 27 axes as stated caveats. The out-of-range `:offset` crash is a **new** finding surfaced by this pass's fuzzing.
