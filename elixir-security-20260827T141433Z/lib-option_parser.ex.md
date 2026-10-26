# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/option_parser.ex
<!-- sha256: a1b789a76464cceab54b8beed828a87b628a6cd731d1be9c528a880f4505895f -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/option_parser.ex
<!-- sha256: a1b789a76464cceab54b8beed828a87b628a6cd731d1be9c528a880f4505895f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28T00:00:00Z (hedge-resolution pass)_

Re-review to discharge the two hedges flagged in the first pass ("did not exhaustively"). Both were resolved with **executed evidence**, not by reading neighbours: (1) `split/1` under malformed UTF-8 was pushed through the real implementation and shown to be byte-preserving with no decode sink; (2) the "in-tree consumers of `:regex`" question was answered by a full-tree search of `lib/`. Both prior findings survive unchanged. No findings dropped.

# Security review — `option_parser.ex`

## 1. Summary
`OptionParser` is safe in its most sensitive area (atom creation is correctly gated behind the documented `allow_nonexistent_atoms:` opt-in — re-confirmed: default-path atom delta = 0 over 3000 distinct unknown switches). Two residual issues stand: parsing a large numeric value for an `:integer`/`:float` switch raises an uncaught `SystemLimitError` instead of returning it as invalid, and the `:regex` switch type compiles an attacker-controlled pattern and hands back a live ReDoS primitive.

## 2. Security

#### `CSCAN-option_parser-numeric-parse-crash`
- **Finding ID:** `CSCAN-option_parser-numeric-parse-crash`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/elixir/lib/option_parser.ex:696-706` (`validate_option/2`, `:integer` and `:float` clauses)
- **Class:** Resource consumption (unbounded numeric parse) / Validation (documented contract violated)
- **Issue:** For `:integer`/`:float` switches, `validate_option/2` calls `Integer.parse(value)` / `Float.parse(value)` on the raw switch value with no length bound. The module documents that a value that "can't be parsed according to the given type … is returned in the invalid options list." But for a sufficiently long numeric value the underlying `:erlang.binary_to_integer/1` raises `SystemLimitError`, which propagates uncaught out of `parse/2`, `parse!/2`, `parse_head/2`, `parse_head!/2` and `next/2`. Sub-limit large values are accepted but cost super-linear CPU/memory (a giant bignum) rather than being rejected.
- **Impact / threat:** A program that feeds `OptionParser` argv derived from untrusted input — e.g. `OptionParser.split(user_string) |> OptionParser.parse(...)`, or an argv list built from a request/config/file — can be crashed by a single ~1.3 MB (integer) / ~2 MB (float) numeric switch value. The caller, relying on the documented `{parsed, args, invalid}` return, will not have wrapped the call in `try/rescue`, so the exception takes down the request/task (DoS). Trust boundary: untrusted string forwarded by the caller into a documented public API. (Not reachable through a normal `mix task` CLI, where the OS caps a single argv element at ~128 KB — which parses in a few ms — hence Low.)
- **Evidence:** `verified` (re-confirmed this pass).
  - `OptionParser.parse(["--limit", "xyz"], strict: [limit: :integer])` → `{[], [], [{"--limit", "xyz"}]}` (documented "invalid" behavior).
  - `OptionParser.parse(["--limit", String.duplicate("9", 1_300_000)], strict: [limit: :integer])` → raises `{SystemLimitError, "a system limit has been reached"}` (re-run this pass). Prior pass also confirmed `:float` with 2 M digits and `OptionParser.next/2`.
  - Sub-limit CPU curve (integer, prior pass): 100k digits=3 ms, 200k=7 ms, 400k=22 ms, 800k=66 ms (super-linear).
- **Suggested fix:** Bound the work before parsing, e.g. treat over-long numeric strings as invalid (`byte_size(value) > limit -> :invalid`), or wrap the `Integer.parse`/`Float.parse` calls so a `SystemLimitError` is caught and mapped to `:invalid`, preserving the documented "goes to the invalid list" contract.

#### `CSCAN-option_parser-regex-untrusted-compile`
- **Finding ID:** `CSCAN-option_parser-regex-untrusted-compile`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `lib/elixir/lib/option_parser.ex:709` (`validate_option/2`, `:regex` clause: `Regex.compile(value, "u")`)
- **Class:** Template/interpolation (regex compiled from external input) / Resource consumption (ReDoS)
- **Issue:** The `:regex` switch type compiles the raw, attacker-controlled switch value into a `Regex`. OptionParser's compile step itself is bounded (pathological patterns either compile instantly or are rejected as `:invalid` by PCRE limits), but the *returned compiled `Regex`* is a live ReDoS primitive handed back to the caller.
- **Impact / threat:** A program declaring a `:regex` switch and parsing untrusted argv gets back a `Regex` the attacker fully controls; when the caller matches it against any non-trivial input, a pattern such as `(a+)+$` yields catastrophic backtracking (DoS). The dangerous compile is done here on external input; the blow-up is realized at the caller's `Regex.match?`. There is no ReDoS-safe compile/match in stdlib, so OptionParser cannot fully neutralize it — but the exposure is worth calling out at the point where the untrusted pattern is turned into an executable matcher.
- **Evidence:** `verified` (re-confirmed this pass). `OptionParser.parse(["--re", "(a+)+$"], strict: [re: :regex])` returns `{[re: ~r/(a+)+$/u], [], []}` (re-run). Prior pass: matching that returned regex against `String.duplicate("a",30) <> "!"` took 76 ms and grows exponentially; compile-time DoS attempts (500k-char literal, 50k groups) were rejected as `:invalid`; a nested-quantifier bomb compiled in 0.16 ms.
- **Suggested fix:** Document on the `:regex` type that the pattern is untrusted-attacker-controllable and callers must not match the result against untrusted data without a size/complexity bound; consider offering an option to cap pattern length.

### Ruled out
- **Atom-table exhaustion (default path)** — By default `to_existing_key/2` (`option_parser.ex:857-863`) uses `String.to_existing_atom/1` rescuing `ArgumentError → nil`; atom creation only happens under `allow_nonexistent_atoms: true`, which routes to `String.to_unsafe_atom/1` (`option_parser.ex:855`) and is explicitly documented as dangerous. *Verified (executed this pass):* 3000 distinct unknown switches with defaults → atom-table delta = 0. Working as documented, gated by explicit opt-in.
  - Axes examined: default vs `allow_nonexistent_atoms: true`; distinct-key volume. | Not examined: none.
- **`split/1` malformed-UTF-8 decode/normalization sink** — *Upgraded from hedged assumption in first pass* (the first pass reasoned rather than executed). *Verified by execution this pass:* `split/1` is byte-oriented (`<<h, t::binary>>` / `<<buffer::binary, h>>`) and performs no UTF-8 decode or normalization. Ran `OptionParser.split(<<0xff,0xfe,"foo",0x20,"bar",0x80>>)` → `[<<255,254,102,111,111>>, <<98,97,114,128>>]`; the output segments concatenate back to the input bytes verbatim (equality check returned `true`). Lone-continuation bytes adjacent to a quote (`<<?", 0xff, 0xfe, ?">>` → `[<<255,254>>]`) and a truncated multibyte tail (`<<"a",0xC3>>` → `[<<97,195>>]`) also pass through byte-preserved without raising or replacement-char substitution. No decode error is swallowed because no decode is attempted.
  - Axes examined: leading invalid bytes, lone continuation byte, truncated multibyte at end, invalid byte inside a quoted run. | Not examined: none.
- **`split/1` quadratic blow-up** — The `<<buffer::binary, h>>` accumulator uses the append-optimized binary path and the repeated `String.trim_leading/2` scans each run of spaces once. *Verified linear (prior pass):* 100k→2 ms … 600k→15 ms for the "word + N spaces + word" case; 800k no-space→4.7 ms.
  - Axes examined: many-spaces run, no-space single token, growth vs length. | Not examined: pathological escaped-quote-heavy input (not a plausible super-linear trigger given the single byte-at-a-time recursion).
- **`split/1` raises on unbalanced quote** — `do_split(<<>>, _, _, marker)` (`option_parser.ex:629`) raises `RuntimeError` on an unterminated quote. *Verified (re-run this pass):* `split(<<?", "abc">>)` raises `"argv string did not terminate properly, a \" was opened but never closed"`. O(1), catchable, syntactic error inherent to a shell-style splitter; not a meaningful DoS.
  - Axes examined: single/double quote, unterminated at EOF. | Not examined: none.
- **`to_argv/2` `List.duplicate(switch, value)` for `:count`** — `value` and `switches` are both caller-supplied data being serialized, not data crossing a trust boundary; parse→to_argv round-trips are symmetric (count = number of occurrences), so no amplification.
  - Axes examined: trust-boundary direction (serialization of caller data). | Not examined: negative/huge `value` passed directly to `to_argv` by a caller — but that is caller-supplied data on the caller's own side of the boundary, not attacker-reachable through parse.
- **`did_you_mean`/`String.jaro_distance` on a long option name** — Only reached in the `parse!`/`parse_head!` error path; cost is linear in the option length times the small caller-defined switch set. No quadratic behavior.
  - Axes examined: reachability (error path only), cost model. | Not examined: adversarial jaro cost on a multi-megabyte option name — reachable only when the caller both uses `parse!` and forwards an untrusted giant option name; bounded by switch-set size, not flagged as a separate low-confidence finding because jaro is linear per comparison.
- **Multi-letter alias expansion recursion (`expand_multiletter_alias` / `next_with_config`)** — Expansion always reduces to single-letter options that hit the `String.length(option) == 1` branch and do not re-expand; recursion terminates and makes progress. *Traced (prior pass)* for ASCII, digit-suffixed (`-n2`), and multi-byte (`-λλ`) inputs.
  - Axes examined: ASCII, digit suffix, multi-byte grapheme. | Not examined: none material (grapheme split guarantees each recursion consumes ≥1 grapheme).

## 3. Risks / caveats
- **Reproductions were run against the toolchain at `/opt/elixir` (Elixir 1.21.0-dev / OTP 29, erts-17.0.2).** Stack traces in the prior pass pointed at `lib/option_parser.ex` at the reviewed line numbers; the loaded module matches this source.
- **The numeric-crash thresholds (~1.3 M digits integer, ~2 M digits float) are VM/`erts`-dependent** (bignum `system_limit`) and may differ on other OTP builds; the *contract violation* (an uncaught `SystemLimitError` where the docs promise an "invalid" entry) holds regardless of the exact threshold.
- **The `:regex` ReDoS is realized in the caller's match, not inside OptionParser.** *Resolved from the first-pass hedge ("I did not survey callers"):* I searched all of `lib/` (`elixir/lib/**/*.ex` and the full tree) for the `:regex` switch type — **no in-tree consumer declares a `:regex` switch**, so no first-party code feeds untrusted patterns into `Regex.compile` via OptionParser. The residual exposure is therefore entirely third-party/external callers using the documented `:regex` type on untrusted argv; the finding stays reported (untrusted input demonstrably reaches `Regex.compile` through a documented public API) at Low/medium because OptionParser's own compile step is bounded and no in-tree caller is affected.

## Did I verify this?
- **Numeric crash (`:integer`/`:float`):** verified — re-ran the 1.3 M-digit integer case → `SystemLimitError`; confirmed the "xyz" invalid path still returns `{[], [], [{"--limit","xyz"}]}`.
- **`:regex` untrusted compile:** verified — re-ran `("--re","(a+)+$")` → returns a compiled `~r/(a+)+$/u`; caller-side blow-up measured in prior pass.
- **`split/1` malformed UTF-8 (was hedged):** verified by execution this pass — byte-preserving, no decode sink, no crash, no replacement char. Hedge discharged.
- **`:regex` in-tree consumers (was hedged):** verified by tree-wide search — none exist. Hedge discharged.
- **Default-path atom safety:** verified — atom-count delta = 0 over 3000 distinct unknown switches.

### Dropped in re-review
None. Both findings and all ruled-out entries are retained; the two first-pass hedges were upgraded to executed/searched verification rather than dropped or downgraded to findings.
