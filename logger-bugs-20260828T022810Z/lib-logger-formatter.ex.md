# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/formatter.ex
<!-- sha256: 945f43206536e9390caee4bd2e9b12b2f100211aab8c6bb7e2fe2f21d08e34a1 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/formatter.ex
<!-- sha256: 945f43206536e9390caee4bd2e9b12b2f100211aab8c6bb7e2fe2f21d08e34a1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28. Reproductions run against the source file compiled directly with `Code.compile_file("lib/logger/formatter.ex")` on Elixir 1.21.0-dev / OTP 29 — not merely the installed build._

## Summary

`Logger.Formatter.prune/1` fails to strip UTF‑16 surrogate code points, so its documented job of removing invalid Unicode code points is not done and the "sanitized" data still cannot be encoded/printed; two lower-severity contract issues also exist in `compile/1` and `system_time_to_date_time_ms/2`. All three are verified with executed evidence against the source under review.

## Bugfixes

#### `CSCAN-formatter-prune-allows-surrogates`

- **Finding ID** — `CSCAN-formatter-prune-allows-surrogates`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `formatter.ex:359`, `Logger.Formatter.prune/1` list clause
- **Class** — Unicode correctness (guard admits invalid code points)
- **Issue** — The clause `def prune([h | t]) when h in 0..1_114_111, do: [h | prune(t)]` keeps any integer up to `0x10FFFF`, but the UTF‑16 surrogate range `0xD800..0xDFFF` are *invalid* Unicode code points that cannot appear in valid UTF‑8. `prune/1` is documented as *"Prunes invalid Unicode code points from lists and invalid UTF-8 bytes"* and is *"Typically called after formatting when the data cannot be printed."* A surrogate in the input list passes the guard and survives untouched, so the value `prune/1` returns still cannot be encoded — defeating the function's entire purpose. (The binary path `prune_binary/2` at `formatter.ex:364-366` is unaffected because `<<h::utf8>>` already rejects surrogate encodings, sending them to the `<<_, t::binary>>` replacement clause; only the list-guard is wrong.)
- **Suggested fix** — Split the guard to exclude the surrogate block: `def prune([h | t]) when h in 0..0xD7FF or h in 0xE000..0x10FFFF, do: [h | prune(t)]` (0x10FFFF == 1_114_111). Surrogates then fall through to `def prune([h | t]), do: [prune(h) | prune(t)]` → `prune(_) -> @replacement`.
- **Impact / threat** — A caller who invokes `prune/1` to make unprintable log data safe still gets data that `:unicode.characters_to_binary/1` rejects (demonstrated below — the pruned value itself is unencodable, so the defect is fully observable on `prune/1`'s own return value, not an assumption about a downstream device write). Log messages/metadata can contain attacker- or peer-influenced code-point lists (e.g. a decoded external string), so this is reachable with untrusted data through the documented public API.
- **Evidence** — `verified` (re-run against directly-compiled source).\
  `F.prune([0xD800, ?a])` → `[55296, 97]` (surrogate not pruned); `:unicode.characters_to_binary(F.prune([0xD800, ?a]))` → `{:error, "", [55296, 97]}` — the returned "sanitized" value fails to encode, in this file's own output.\
  Boundary sweep (compiled from source):
  ```
  D7FF: pruned=[55295] encodes?=true
  D800: pruned=[55296] encodes?=false
  DBFF: pruned=[56319] encodes?=false
  DC00: pruned=[56320] encodes?=false
  DFFF: pruned=[57343] encodes?=false
  E000: pruned=[57344] encodes?=true
  ```
  The whole `0xD800..0xDFFF` range survives and is unencodable, confirming the guard should exclude it.

#### `CSCAN-formatter-compile-leading-dollar-literal`

- **Finding ID** — `CSCAN-formatter-compile-leading-dollar-literal`
- **Severity** — Low
- **Confidence** — high
- **Location** — `formatter.ex:442-448`, `Logger.Formatter.compile/1` binary clause
- **Class** — Missing clause / pattern match on the wrong shape (assumes every `"$"`-prefixed part came from the regex split)
- **Issue** — After `Regex.split(~r/(?<head>)\$[a-z]+(?<tail>)/, ...)`, the code assumes any part starting with `"$"` is a `$name` pattern: `case part do "$" <> code -> compile_code(String.to_unsafe_atom(code)) ... end`. But a literal segment can also start with `$` when the `$` is *not* followed by `[a-z]+` and sits at the start of a split segment. `compile("$5 today")` is never split, so the whole string enters the `"$" <> code` branch with `code = "5 today"`, producing `String.to_unsafe_atom("5 today")` → `compile_code/1`'s `is_atom` clause raising `ArgumentError: "$5 today is an invalid format pattern"`. The same `$5` mid-string (`"cost $5 today"`) is correctly treated as a literal. The behavior is position-dependent and the error names the entire remaining string rather than a pattern, telling the caller nothing actionable.
- **Suggested fix** — Only treat a part as a pattern when it matches a real `$name`, e.g. guard the branch with `part` matching `~r/^\$[a-z]+$/` (or re-check `code =~ ~r/^[a-z]+$/`), otherwise keep `part` as a literal — matching the mid-string behavior.
- **Impact / threat** — A developer whose format string legitimately begins with a literal `$` (currency, shell-like text) gets a confusing startup-time `ArgumentError` naming their whole format string as "an invalid format pattern". Compile-time operator config, not untrusted runtime input, hence Low.
- **Evidence** — `verified` (re-run against directly-compiled source). `F.compile("$5 today")` → `{:raise, "$5 today is an invalid format pattern"}`; `F.compile("cost $5 today")` → `["cost $5 today"]`.

#### `CSCAN-formatter-negative-systime-ms`

- **Finding ID** — `CSCAN-formatter-negative-systime-ms`
- **Severity** — Low
- **Confidence** — high
- **Location** — `formatter.ex:396,404`, `Logger.Formatter.system_time_to_date_time_ms/2`
- **Class** — Contract violation (return value outside declared `@type`)
- **Issue** — `micro = rem(system_time, 1_000_000)` then `div(micro, 1000)`. For a negative `system_time` (a valid `integer()` per `@spec system_time_to_date_time_ms(integer(), boolean())`, i.e. a pre-1970 timestamp), `rem` yields a negative remainder, so the milliseconds field is negative — e.g. `-500`. The declared return type is `date_time_ms :: {date, {0..23, 0..59, 0..59, 0..999}}`, which the value `{{1969,12,31},{23,59,59,-500}}` violates. Feeding that tuple to `format_time/1` (`formatter.ex:372-373`) renders the ms via `pad3(-500)` → `pad3/1`'s `int < 100` clause → `"00-500"`, i.e. garbage like `23:59:59.00-500`.
- **Suggested fix** — Normalize the sub-second part to be non-negative, e.g. `micro = Integer.mod(system_time, 1_000_000)` (or `rem(rem(system_time, 1_000_000) + 1_000_000, 1_000_000)`) before `div(micro, 1000)`.
- **Impact / threat** — Only reachable with a negative timestamp; the real `format/2` event path guards `time >= 0`, so live log events never hit it. A custom-formatter author calling this documented helper with a pre-epoch timestamp gets a type-contract-violating tuple and malformed time output. Low.
- **Evidence** — `verified` (re-run against directly-compiled source). `F.system_time_to_date_time_ms(-500_000, true)` → `{{1969, 12, 31}, {23, 59, 59, -500}}`.

### Ruled out

- **`compile/1` `String.to_unsafe_atom` atom-table growth**\
  Axes examined: input-provenance (compile-time operator config vs runtime), reuse (atoms created once per `compile/1` at handler setup). | Not examined: none.\
  The atoms come from operator-supplied compile-time format strings compiled once; invalid ones raise before any reuse. Not attacker-driven, not a runtime path.

- **`colorize/1` `Map.fetch!(colors, level)` KeyError on `:warn`**\
  Axes examined: legacy `:warn` level (tried — raises `KeyError`), all 8 OTP levels (tried — format cleanly). | Not examined: none.\
  A well-formed `:logger.log_event` carries OTP levels (`:warning`, …), all present in the colors map. Reaching the KeyError requires an event that violates the `:logger.log_event` level contract, and only via the `@doc false` `format/2`. Out of scope (caller contract violation).

- **`metadata(:domain, [head | tail])` → `Atom.to_string` on non-atom (`ArgumentError`)**\
  Axes examined: mixed `[:a, "b"]` domain list (tried via `format/5` — raises). | Not examined: none.\
  `:domain` is `[atom()]` per the `:logger` metadata contract; a mixed list violates the caller's contract. Out of scope.

- **`metadata/2` dropping charlist values** (`[foo: ~c"hi"]` → `""`)\
  Axes examined: charlist metadata value (tried — dropped). | Not examined: none.\
  The module doc explicitly excludes charlists ("Implement the `String.Chars` protocol (except for charlists)"). Documented behavior, not a bug.

- **`add_reset/1` CRLF/nested-chardata handling**\
  Axes examined: binary tail, `[?\r,?\n]` tail, `[?\n]` tail, integer tail, nested-list tail (all tried via `format/2` with colors). | Not examined: none.\
  Reset is consistently placed before the trailing newline in every traced case. No defect.

- **`compile/1`, `format/5`, `format_time/1`, `format_date/1`, `truncate/2` doctests/examples**\
  Axes examined: published doctest inputs (all tried and matched: compile split, `"[info] hello"`, truncation marker `"hello (truncated)"`). | Not examined: none.\
  No code/doc mismatch.

## Risks / caveats

- Only `formatter.ex` was reviewed. `Logger.Utils.truncate_n/2`/`fix_binary/2` (invoked by `truncate/2` at `formatter.ex:348`) and `Logger.Utils.scan_inspect/4` live in `utils.ex`; they are collaborators and belong to that file's own review — in particular the surrogate-in-truncation edge and the binary-branch non-rescue in `wrapped_truncate/2`. This is a scope referral, not an unverified claim about `formatter.ex`.
- **(Resolved from first pass)** The prior report hedged that the surrogate failure "occurs outside this file." That is no longer a hedge: `:unicode.characters_to_binary(F.prune(...))` returning `{:error, …}` is executed evidence that `prune/1`'s *own return value* violates its documented contract inside this file. The downstream device-write is merely one consumer; the defect is proven on the return value itself.
- **(Resolved from first pass)** The prior report hedged that reproductions "assumed" the installed build matched the source. Re-run: all three reproductions were executed against the source file compiled directly via `Code.compile_file("lib/logger/formatter.ex")`, so the outputs above reflect the exact code under review, not an assumed-matching build.

## Did I verify this?

- **prune surrogate**: Yes — executed against source-compiled module; `[0xD800, ?a]` survives and fails `:unicode.characters_to_binary`, plus full `0xD800..0xDFFF` boundary sweep.
- **compile leading-dollar literal**: Yes — executed; `"$5 today"` raises, `"cost $5 today"` is a literal.
- **negative systime ms**: Yes — executed; returns `{{1969,12,31},{23,59,59,-500}}`, outside the declared `date_time_ms` type.
- **All ruled-out entries**: Each axis listed under "Axes examined" was actually executed (tried), and each entry's "Not examined" is `none`. None were cleared by reading alone.

## Dropped in re-review

None. All three findings from the first pass are preserved and re-verified against the directly-compiled source. No hedged assumption required upgrading to a low-confidence finding, because each was resolvable by executing the axis (recompiling from source and re-running) rather than by asserting an invariant.
