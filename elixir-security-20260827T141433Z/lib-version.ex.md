# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/version.ex
<!-- sha256: 8336eb75fc9875a07d345f4ab112ee076f94e0bb8dc437c1822f88e4756299da -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:52:24Z_

Both findings are verified and precisely bounded. The crash is specific to *invalid* UTF-8 bytes in the requirement lexer; the version parser (`parse`/`parse!`/`compare`) handles all bad input gracefully. Here is the report.

---

## Summary

`Version` is a pure semver parser/matcher with no code-execution, atom-creation, file, or command sinks; the only security-relevant issues are in the requirement **lexer**, which crashes with an undocumented exception on invalid-UTF-8 input and amplifies each `!` byte into a stacktrace-bearing warning (log-flood/CPU DoS).

## Security

#### `CSCAN-version-requirement-bang-warn-amplification`
- **Finding ID:** `CSCAN-version-requirement-bang-warn-amplification`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `lib/version.ex:564-572` (`Version.Parser.lexer/3`, the `"!="` and `"!"` clauses), reachable via `Version.parse_requirement/1`, `parse_requirement!/1`, `match?/2,3`
- **Class:** Resource consumption
- **Issue:** During lexing, every `!=` and every bare `!` byte calls `IO.warn/1`, which captures and formats a full stacktrace and writes a multi-line message to stderr. The warning fires *per occurrence* with no de-duplication and *before* the requirement is structurally validated, so it fires even on input that is ultimately rejected as `:error`. A requirement consisting only of `!` characters therefore produces one stacktrace-bearing 3-line warning per input byte.
- **Impact / threat:** An attacker who can influence a requirement string that reaches this parser (e.g. a package registry such as Hex parsing version requirements from uploaded package metadata, a lockfile, or any app calling `Version.match?/3` on a user-supplied requirement) can force linear-amplified log flooding and CPU burn: ~57 µs and 3 stderr lines per byte. A ~1 MB payload yields ~1 minute of CPU and ~3 million log lines, which can fill disk on any service that captures stderr/logs — an availability DoS across a trust boundary (untrusted requirement text → compiler-adjacent library).
- **Evidence:** `verified`.
  - `Version.parse_requirement(String.duplicate("!", 20_000))` → `RESULT=:error TIME_MS=1568.08`; `grep -c deprecated` on captured stderr = `20000` (i.e. 20 000 warnings, 3 lines each, before the `:error`).
  - Scaling run: `n=2000 time_ms=115.29`, `n=4000 time_ms=230.19`, `n=8000 time_ms=455.6` (linear, ~57 µs/byte), total `deprecated` lines = `14000`.
  - Each warning includes a formatted stack frame block, e.g. `(elixir 1.21.0-dev) lib/version.ex:570: Version.Parser.lexer/3` — confirming per-byte stacktrace capture.
- **Suggested fix:** Emit the deprecation warning at most once per `parse_requirement` call (e.g. record that a `!`/`!=` was seen and warn a single time after lexing), or drop the warning on the `!` path and reject in `revert_lexed`. At minimum, do not capture a stacktrace per token.

#### `CSCAN-version-requirement-lexer-utf8-crash`
- **Finding ID:** `CSCAN-version-requirement-lexer-utf8-crash`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/version.ex:578-584` (`Version.Parser.lexer/3`; the `<<char::utf8, rest::binary>>` clause plus missing invalid-byte fallback), reachable via `Version.parse_requirement/1`, `parse_requirement!/1`, `match?/2,3`
- **Class:** Unclassified (decoder crash on malformed input / contract violation)
- **Issue:** The lexer consumes the buffer only via `<<char::utf8, rest::binary>>` and a `<<>>` terminator. A byte that is not valid UTF-8 (e.g. `0xFF`, or a truncated multi-byte lead such as `0xC3`) matches none of the operator/whitespace/`utf8`/empty clauses, so `lexer/3` raises `FunctionClauseError`. This contradicts the documented contract: `parse_requirement/1` is specced `{:ok, Requirement.t()} | :error`, and `parse_requirement!/1`/`match?/3` are documented to raise `Version.InvalidRequirementError`. The sibling version parser (`parse_version` via `String.split`) handles the identical bad input gracefully, so this is an inconsistency in the module rather than intended behavior.
- **Impact / threat:** A caller that treats a requirement string as untrusted decoder input (package metadata, registry response, lockfile, user input) and matches on `:error` or rescues `Version.InvalidRequirementError` will instead take an uncaught `FunctionClauseError`, crashing the calling process on malformed bytes. Impact is bounded (an exception, no memory/CPU blowup), hence Low.
- **Evidence:** `verified`.
  - `Version.parse_requirement(<<0xFF>>)` → `RAISED: no function clause matching in Version.Parser.lexer/3` (`FunctionClauseError`), whereas `Version.parse(<<0xFF>>)` → `:error`.
  - `Version.match?("1.2.3", <<0xFF>>)` → `UNEXPECTED FunctionClauseError` (documented type is `Version.InvalidRequirementError`); same for `parse_requirement!/1`.
  - Trigger is specifically *invalid* UTF-8: `Version.parse_requirement("é")` → `:error` (graceful), `Version.parse_requirement(<<0xC3>>)` → `FunctionClauseError`.
- **Suggested fix:** Add a terminal fallback clause to the lexer for non-UTF-8 input that fails the parse, e.g. `defp lexer(<<_, _::binary>>, _buffer, _acc), do: :__invalid__` and have `parse_requirement` map it to `:error` (or validate `String.valid?/1` up front and return `:error`), mirroring `parse_version`'s graceful rejection.

### Ruled out
- **Quadratic/exponential blowup in the lexer** — the char-append `<<buffer::binary, char::utf8>>` relies on the BEAM binary-append optimization; measured linear: `n=20000→0.09 ms … n=160000→0.76 ms`. `verified`, not a finding.
- **`compare/2`, `parse/1`, `parse!/1` on malformed/invalid-UTF-8 input** — all route through `parse_version`, which returns `:error` (`Version.parse(<<0xFF>>)` → `:error`, verified), so `parse!`/`compare` raise the *documented* `InvalidVersionError`. No crash, no warn-flood. `verified`.
- **Atom exhaustion** — the lexer produces only the fixed compile-time operator atoms; numeric components go through `:erlang.binary_to_integer` (bounded to 14 digits, `require_digits`/`@max_numeric_component_digits`). No external-string-to-atom conversion. `verified` by inspection of all clauses.
- **Large-integer / numeric DoS** — every numeric component (major/minor/patch and numeric pre-release identifiers) is rejected past 14 digits before integer conversion (`require_digits`, `convert_parts_to_integer` byte-size guard), so no bignum blowup. `asserted` from the guards (bound is enforced structurally).
- **`to_string/1` round-trip / interpolation** — interpolates only integer/binary struct fields into a version string with no downstream interpreter; a non-int/non-binary `:pre` element would raise, but only via directly constructing the struct (explicitly documented as unsupported / out of scope).
- **`Requirement`/`Version` struct-field trust, protocol dispatch** — `@derive {Inspect, …}` and the `String.Chars`/`Inspect` impls emit only already-parsed fields; no secret-bearing data, no dispatch on external type/module names.

## Risks / caveats
- The invalid-UTF-8 crash requires a byte string that violates the `String.t()` (valid-UTF-8) type contract. I report it rather than ruling it out because (a) requirement strings are a classic untrusted-decoder input, (b) the documented failure mode (`:error` / `InvalidRequirementError`) is bypassed by an undocumented `FunctionClauseError`, and (c) the sibling `parse_version` defends against the exact same input — so this is an internal inconsistency, not merely caller misuse. A reviewer who treats "input must be valid UTF-8" as an absolute precondition may downgrade it to out-of-scope.
- The warn-amplification DoS is **linear** in input size (verified), not super-linear; its severity depends on whether a real deployment (a) parses attacker-influenced requirement strings and (b) persists stderr/log output. Hex-style registries that parse requirements from uploaded packages are the strongest realistic consumer; a purely developer-local `mix.exs` use is low-risk.
- I did not audit callers of `Version` (Mix/Hex) themselves — only what is reachable through this file's documented public API. Whether malformed requirement bytes actually reach `parse_requirement` in a given pipeline is caller-dependent and unverified here.
- All timing figures were measured in this sandbox (Elixir 1.21.0-dev / OTP 29, JIT) with stderr piped to a file; absolute numbers will vary with hardware and log destination, but the per-byte amplification and the crash behavior are structural and reproducible.

