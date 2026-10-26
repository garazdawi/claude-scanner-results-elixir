# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/naive_datetime.ex
<!-- sha256: b992310954d513802dd463e9b055d77e896766e5634623b49507a8331b4d5054 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:09:40Z_

I have read the entire file and empirically tested every plausible sink. Here is the report.

# Security Review — `naive_datetime.ex`

## 1. Summary
`NaiveDateTime` is a pure, numeric date/time struct module whose only dynamic behaviour is contract-bound calendar dispatch and delegated ISO-8601 parsing; I found no security defect reachable through its documented public API.

## 2. Security

`No findings`

This module contains none of the dangerous sink families: no `Code.eval`/`compile`/`apply` on input, no `*_to_atom`/`Module.concat`, no `System.cmd`/`os.cmd`, no `File`/`Path`/archive operations, no `binary_to_term`, no `:crypto`, no ETS/persistent_term/process-dictionary writes, no processes or check-then-act races, and no `Regex` built from input. Its inputs are integers, an Erlang datetime tuple, a `Duration`, and an ISO-8601 string; the string is parsed by `Calendar.ISO.parse_naive_datetime/1` (a different file, out of scope), and this module only assembles/validates the result.

### Ruled out

- **Behaviour dispatch on the `calendar` field** (`to_iso_days`/`from_iso_days`/`to_string`/`to_gregorian_seconds`/`shift`/etc., e.g. `calendar.naive_datetime_to_string(...)`). *Basis: verified.* The dispatched atom comes from the struct's `calendar` field. Reaching it with a hostile module requires the caller to hand-build a struct/map with a non-calendar `calendar:` value — a violation of the `Calendar.naive_datetime()` contract, and the excluded "caller passes contract-violating args" case. Even with full control of the atom, the only invokable names are calendar-callback names (`naive_datetime_to_iso_days/7`, `valid_date?/3`, …) that no dangerous OTP module exposes; I confirmed the worst outcome is a plain crash: `NaiveDateTime.to_string(%{calendar: :os, ...})` → `UndefinedFunctionError`, not code execution.
- **Resource consumption via large `add/3`, `diff/3`, `shift/2` amounts.** *Basis: verified.* All paths are BEAM bignum arithmetic with no superlinear amplification. `add(~N[0000-01-01 00:00:00], 10^40, :second)` completes in 0 ms; `add(..., 100_000_000_000_000, :day)` (×86400) is 0 ms; `shift(year: 1_000_000_000)` is 0 ms. Output year digit-count tracks input size, so there is no cheap way to force a huge allocation.
- **DoS through the `from_iso8601/2` parse entry point (untrusted string).** *Basis: verified.* A 5,000,000-digit fractional part parses in 11 ms (truncated to 6 digits); a 5,000,000-digit year returns `{:error, :invalid_format}` in 0 ms. No pathological blowup.
- **Round-trip integrity of `from_iso8601` ↔ `to_iso8601`.** *Basis: verified.* Values round-trip consistently; the only asymmetries (`,`→`.` decimal separator, truncation to 6 fractional digits, offset discarded) are documented and semantics-preserving. This module is not used as a signature/cache-key canonicalizer, so the asymmetry has no trust impact.
- **Log/format-string injection via error messages** (`from_iso8601!`, `from_erl!`, `new!`, `convert!`, `diff`, `local_now`). *Basis: verified.* Every untrusted value is passed through `inspect/1` before interpolation. `from_iso8601!("evil\nINJECTED \e[31m")` yields `... "evil\nINJECTED \e[31m" ...` — newline/ANSI are escaped, not emitted raw.
- **Validation predicates (`new/8`, `from_erl/3`).** *Basis: verified via module behaviour/doctests.* Range/leap-year checks are delegated to `calendar.valid_date?/valid_time?` and correctly return `{:error, :invalid_date|:invalid_time}` (e.g. `new(2001,2,29,...)` → `{:error, :invalid_date}`); no invalid struct escapes as `{:ok, _}`.
- **Atom creation from the `unit` argument** (`add/3`, `diff/3`). *Basis: verified by reading.* Units are compared against fixed atom literals with `in ~w(...)a` / clause matching; no `String.to_atom` conversion occurs, so no atom-table growth.
- **Dead defensive code.** *Basis: verified by reading.* The `unit not in ~w(...)a` guards in `add/3` and `diff/3` are reachable (pass an unsupported unit) and raise `ArgumentError`; not unreachable.

## 3. Risks / caveats
- **Out of scope, by design:** the actual byte-level ISO-8601 parser (`Calendar.ISO.parse_naive_datetime/1`) and the calendar-callback implementations (`Calendar.ISO.*`, third-party calendars) live in other files. This review assumes those honour their contracts; a parser bug there would surface through `from_iso8601/2` but is not this file's defect.
- **Untested extreme:** formatting (`to_string`/`to_iso8601`/`Inspect`) of a year with, say, 10⁹ digits would be slow — but producing such a year requires an input integer of comparable size, so there is no attacker amplification and I did not treat it as a finding.
- **Contract reliance:** several clauses match on a bare `%{calendar: ...}` map rather than `%NaiveDateTime{}`, so callers who forward attacker-controlled *structs* (e.g. from `:erlang.binary_to_term` on untrusted bytes) could steer calendar dispatch — but that is the deserializer's boundary, not this module's, and the resulting call is crash-only as shown above.

