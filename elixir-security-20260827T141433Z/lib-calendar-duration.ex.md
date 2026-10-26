# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/duration.ex
<!-- sha256: d7b3ed9db21577f200a4d8bbe62e635aa1186c3194624b694304f248badf98c0 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/duration.ex
<!-- sha256: d7b3ed9db21577f200a4d8bbe62e635aa1186c3194624b694304f248badf98c0 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

## Summary

`Duration` is a small, pure data-manipulation module whose only untrusted-input entry points are `from_iso8601/1` and `from_iso8601!/1`; the parser they delegate to lives in `Calendar.ISO` (`iso.ex`). The sole issue local to this file is that `from_iso8601!/1` echoes the raw, unescaped input string into its exception message (verified by execution). In this re-review I read the delegated parser and executed adversarial inputs to convert the first-pass scoping caveats into either verified dismissals or, where an axis could not be tried, an explicitly-labelled low-confidence entry. The result: no new findings; the parser hedges resolve to `verified` by execution.

## Security

#### `CSCAN-duration-iso8601-error-message-injection`
- **Finding ID:** `CSCAN-duration-iso8601-error-message-injection`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `lib/elixir/lib/calendar/duration.ex:411`, `from_iso8601!/1`
- **Class:** Template / interpolation (log / terminal injection)
- **Issue:** On a parse failure, the untrusted input `string` is interpolated **raw** into the `ArgumentError` message: `~s/failed to parse duration "#{string}". reason: #{inspect(reason)}/`. Note the asymmetry — `reason` is passed through `inspect/1` (which escapes control bytes), but `string` is not. Control characters in the input (ANSI escape sequences, newlines, carriage returns) are carried into the message verbatim.
- **Impact / threat:** `from_iso8601!/1` is a documented public API commonly fed a duration value that originated from user/network input (e.g. an HTTP parameter). On malformed input it raises, and the resulting message routinely reaches two interpreters that act on those bytes: an uncaught exception is printed to **stderr (a terminal)**, where the embedded ANSI escapes execute (screen manipulation, colored/forged text); and error messages are commonly written to **logs**, where an embedded newline lets an attacker forge an additional, attacker-controlled log line. The trust boundary crossed is external input → error string → terminal/log sink. Impact is limited to display/log integrity (spoofing/forgery), not code execution — hence Low.
- **Evidence:** `verified`.
  Command:
  ```
  elixir -e 'try do Duration.from_iso8601!("\e[31;1mFORGED ERROR: system compromised\e[0m\nX") rescue e in ArgumentError -> IO.puts("raw message bytes: " <> inspect(Exception.message(e), limit: :infinity)) end'
  ```
  Output (raw ANSI `\e[31;1m` and the `\n` survive into the message):
  ```
  raw message bytes: "failed to parse duration \"\e[31;1mFORGED ERROR: system compromised\e[0m\nX\". reason: :invalid_duration"
  ```
- **Suggested fix:** Inspect the string as is already done for `reason`:
  ```elixir
  raise ArgumentError, ~s/failed to parse duration #{inspect(string)}. reason: #{inspect(reason)}/
  ```
  `inspect/1` escapes control characters and quotes the value, neutralizing ANSI/newline injection.

### Ruled out

- **`from_iso8601/1` violating its `{:ok,_} | {:error,_}` contract by raising instead of returning `{:error,_}`.**
  - *Axes examined:* fractional precision > 6 (`"PT4.1234567890123S"`, `"PT4.000000000S"`); negative fraction (`"PT-0.500S"`, `"-PT4.5S"`); comma-decimal (`"PT4,5S"`); huge integer field (400-digit year); empty/degenerate fractions (`"PT4.S"`, `"PT.5S"`); fraction on a date component (`"P0.5Y"`); full mixed duration; leading `+`. *Not examined:* none.
  - Every input above returned `{:ok, …}` or `{:error, …}` — none raised (executed batch, output captured this pass). The mechanism is now confirmed by reading the delegated parser: `parse_microsecond/3` (`iso.ex:2063-2064`) hard-caps `precision` at 6 and rejects a zero-length fraction (`iso.ex:2069`), so the tuple reaching `new!` always has `precision ∈ 1..6`, which `validate_unit!/1`'s `precision in 0..6` guard (`duration.ex:218`) accepts; integer fields come only from `Integer.parse` and are always integers, accepted by `validate_unit!/1` at `duration.ex:233`. Thus `new!` cannot raise on parser output. *Basis: verified — executed the axis batch and read `Calendar.ISO.parse_duration` and helpers.*

- **Atom exhaustion in `new!/1` / `from_iso8601/1`.** No `String.to_atom`/`binary_to_atom`/`Module.concat` in this file; the unit keys reaching `new!` are the fixed atoms returned by `find_date_unit/2` (`iso.ex:767-771`) and `find_time_unit/2` (`iso.ex:773-776`) — literal `:year`/`:month`/`:week`/`:day`/`:hour`/`:minute`/`:second`/`:microsecond`, no dynamic atom construction anywhere in the parser — or supplied by the caller, and `validate_unit!/1` allow-lists them (unknown units raise rather than atomize). *Basis: verified by reading this file and the parser's complete unit-resolution clause set (no dynamic-atom sink exists to have an untried axis).*

- **Unbounded-integer / bignum DoS reachable from `from_iso8601`.**
  - *Axes examined:* 400-digit integer field (executed; parsed and returned `{:ok, …}` with no perceptible cost). *Not examined:* multi-megabyte digit runs (the potentially superlinear `Integer.parse`/`binary_to_integer` cost lives in `Calendar.ISO`, `iso.ex:722`/`2073`, not in this file — I did not benchmark that scale).
  - Within `duration.ex` there is no small-input→large-work amplification: `new!/1` does O(units ≤ 8) validation + `struct!`, and `add`/`subtract`/`multiply`/`negate`/`to_string`/`to_iso8601` are all cost-proportional to the integers the caller already holds. The parse cost, if any, originates in the sibling `iso.ex` and should be reviewed where it lives. *Basis: verified for the file boundary by reading; the one benchmark axis tried was cheap, larger-scale cost is external and stays on `Not examined`.*

- **Arithmetic API (`add`, `subtract`, `multiply`, `negate`).** Pure integer arithmetic on struct fields; Elixir integers are arbitrary-precision (no overflow), and results scale with inputs. No trust boundary, no sink. *Basis: verified by reading.*

- **`to_string/2` opts (`:units`, `:separator`) and `@derive {Inspect}`.** Caller-supplied formatting values reach `IO.iodata_to_binary` only; a non-iodata value fails as caller misuse, and durations hold no secrets, so `Inspect` derivation leaks nothing. *Basis: verified by reading.*

### Did I verify this

- **The one finding is executed, not asserted.** The raw-byte survival into `ArgumentError.message` was reproduced against the `elixir` built from the source under review; the ANSI `\e[31;1m` and `\n` are present in the message bytes as quoted.
- **The first-pass scoping hedges are now resolved by reading the neighbour and running inputs.** I read `Calendar.ISO.parse_duration/1` and its helpers (`parse_duration_date/4`, `parse_duration_time/4`, `find_date_unit/2`, `find_time_unit/2`, `parse_microsecond/1,3`) and executed a 12-input adversarial batch. This upgraded the "does `from_iso8601/1` honor its `{:ok,_}|{:error,_}` contract" caveat from a single-example check to a multi-axis executed dismissal (`Not examined: none`).
- **One axis genuinely remains external and untried:** the cost of `Integer.parse`/`binary_to_integer` on multi-megabyte digit runs. That code is in `iso.ex`, not this file; I benchmarked only up to 400 digits (cheap). It stays on `Not examined` for the bignum entry and should be audited in `iso.ex`.
- **Environmental note:** the VM emitted a latin1 native-encoding warning; I ran with `ELIXIR_ERL_OPTIONS="+fnu"` where relevant. This does not affect the byte-level results quoted.

### Dropped in re-review

- None. Every prior finding and every prior ruled-out entry is preserved. Two entries (contract-violation and bignum-DoS) gained an explicit `Axes examined … | Not examined …` field: the contract entry moved to `Not examined: none` on executed evidence; the bignum entry retains one honestly-untried external axis. No entry was upgraded to a finding, because the executed axes confirmed no contract violation or raise path in this file.
