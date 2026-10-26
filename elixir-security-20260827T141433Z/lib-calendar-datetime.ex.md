# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/datetime.ex
<!-- sha256: 9de34ed2985f7b3684550ea61a75462698325f620ec1f7b7c32e43195a94fe0f -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/datetime.ex
<!-- sha256: 9de34ed2985f7b3684550ea61a75462698325f620ec1f7b7c32e43195a94fe0f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (Elixir 1.21.0-dev / OTP 29, built from the tree under review). Every "outside this file" hedge from the first pass was re-checked by pushing hostile values through the real neighbour implementations and comparing returns — not by reading them._

---

## Summary

`DateTime` is a pure calendar-arithmetic module with no code-eval, atom-creation, filesystem, OS, network, crypto, or shared-mutable-state sinks. Its only untrusted-data entry points (`from_iso8601`, `from_unix`, `from_gregorian_seconds`, and the `calendar`/`time_zone_database` polymorphic fields) degrade gracefully under executed hostile input. I found no actionable security issues. Each first-pass dismissal that leaned on downstream behaviour has now been re-run against the actual downstream code.

## Security

No findings.

### Ruled out

- **Behaviour dispatch on the `calendar` field** (`to_string/1`, `to_iso8601/1`, `to_unix/1`, `compare/2`, `diff/3`, `Inspect`/`String.Chars`, `convert/2`, etc.). These invoke `calendar.datetime_to_string(...)`, `calendar.naive_datetime_from_iso_days(...)`, `calendar.day_rollover_relative_to_midnight_utc()` and friends, where `calendar` comes from the passed map — documented `Calendar.datetime()` polymorphism, i.e. an arbitrary-MFA gadget *if* the attacker injects a live module atom.
  Basis: `verified`.
  Axes examined: untrusted **string** in `calendar` (what untrusted JSON yields); **atom** that names no module.
  Not examined: none.
  Evidence I ran: `DateTime.from_unix(0, :second, "Elixir.Foo")` raises `ArgumentError` ("you attempted to apply a function named :day_rollover_relative_to_midnight_utc on \"Elixir.Foo\" … make sure the module is an atom") — a binary is not callable as a module, so **no dispatch to attacker-controlled code occurs**. (First pass predicted `{:error, :badarg}`; the actual result is a clean `ArgumentError` — same security conclusion, corrected wording.) Reaching the gadget still requires a real, existing module atom in `calendar`, which means an upstream `String.to_atom(user_input)`; that atomization is the actual vulnerability and does not live in this file. Same reasoning rules out `time_zone_database.*` dispatch (an explicit behaviour-module argument / configured default, never derived from untrusted data). Retained as an accepted-design caveat below.

- **`from_iso8601/1,2,3` on hostile strings** (the one true network/disk entry point). This module only builds a struct from the tuple returned by `Calendar.ISO.parse_utc_datetime/2` and calls `convert/2`.
  Basis: `verified`.
  Axes examined: non-datetime text, empty string, 100-digit year, out-of-range fields, invalid UTF-8 bytes; and the `calendar`-argument type axis.
  Not examined: none.
  Evidence I ran: `"garbage"`, `""`, `String.duplicate("9",100)<>"-01-01T00:00:00Z"`, `"2015-13-99T99:99:99Z"`, `<<255,254>>`, `<<0,1>>` all return `{:error, :invalid_format | :invalid_date}` with no crash. `from_iso8601(str, "Elixir.Foo")` raises `FunctionClauseError` (the `calendar` param guard requires an atom) — a caller contract violation, not a hostile-input crash. The byte-level parser (`Calendar.ISO.parse_utc_datetime/2`) absorbs the malformed input and returns an error tuple that this file threads through `with`.

- **Large-integer resource consumption** (`from_gregorian_seconds/1,3`, `from_unix/1,2,3`, `add/3`, `diff/3`, `to_unix/2`).
  Basis: `verified`.
  Axes examined: extreme-magnitude integer seconds; Unix-time range boundaries (positive and negative).
  Not examined: none.
  Evidence I ran: `from_gregorian_seconds(10^30)` yields year `31688738506811431594680` with no crash (BEAM bignums; cost linear in input digits, no small-input→large-output amplification). `from_unix(253_402_300_800)` and `from_unix(-377_705_116_801)` both return `{:error, :invalid_unix_time}` — the range check in `Calendar.ISO.from_unix/2` fires before any allocation.

- **`diff/3` / `add/3` unit-guard gap.** The guard `if not is_integer(unit) and unit not in ~w(second millisecond microsecond nanosecond)a` deliberately lets *integer* units through, including `0` and negatives, deferring their validation to `System.convert_time_unit/3`.
  Basis: `verified`.
  Axes examined: `unit = 0` and `unit = -5` through both `diff` clauses (the `:microsecond` naive-diff path **and** the `offset_diff` path, which calls `System.convert_time_unit` a second time) and through `add`; division sites (`:day`/`:hour`/`:minute` clauses `div` by hard-coded constants, never by `unit`).
  Not examined: none.
  Evidence I ran: `DateTime.diff(dt1,dt2,0)`, `DateTime.diff(dt1,dt2,-5)`, `DateTime.add(dt1,10,0)`, `DateTime.add(dt1,10,-5)`, and the offset-path `DateTime.diff(amt,war,0)` all raise `ArgumentError` with the identical text produced by `System.convert_time_unit(5,:second,0)`/`(…,-1)` — confirming the rejection comes from `System.convert_time_unit`, not from this file's own guard, and that it fires before any arithmetic. No division-by-zero is reachable; the only `div` operations use constant denominators. Non-positive integers also violate the `System.time_unit()` spec, so this is contract-violation territory regardless.

- **Error-message interpolation** (`"...in time zone #{time_zone}..."`, `"invalid Unix time #{integer}"`, `"cannot shift #{inspect(datetime)}..."`, etc. in `new!`, `from_naive!`, `shift_zone!`, `now!`, `from_unix!`, `add`, `shift`, `convert!`). I read every raise site in the full file: all land in `ArgumentError` messages, never a downstream interpreter (SQL/shell/HTML/regex); `String.Chars`/`inspect` of a `time_zone` string is inert. No injection sink.
  Basis: `asserted` (this is an inline property of this file — no neighbour axis to try). Confidence for the corresponding dismissal: high.

- **`from_map/1` / `truncate/2` on partial maps** — a missing key raises `KeyError`; a non-2-tuple `microsecond` raises a match error. Pure caller contract violations, explicitly out of scope. Basis: `asserted`.

- **`to_gregorian_seconds/1` → `seconds_from_day_fraction/1` `div/2`** — a zero denominator would require a custom `calendar` module returning a `{_, 0}` day-fraction, i.e. attacker-supplied code already executing. Out of scope. Basis: `asserted`.

- **`to_iso8601`/`from_iso8601` round-trip** — asymmetries change a *datetime's* value, not a security token, cache key, or signature; no trust decision rides on the round-trip. Not a security boundary. Basis: `asserted`.

## Risks / caveats

- **The `calendar`/`time_zone_database` dispatch is an accepted design gadget.** It is safe *given* the ecosystem invariant that these fields hold trusted module atoms and untrusted input is never atomized. Executed evidence confirms a string in `calendar` cannot invoke code (it raises on `apply`); the gadget requires a genuine module atom. If a downstream caller ever does `String.to_atom(user_input)` before populating `calendar`/`time_zone`, that caller — not this file — turns the dispatch into arbitrary-module invocation.
- **Neighbour modules were exercised, not audited line-by-line.** `Calendar.ISO.parse_utc_datetime/2`, `Calendar.ISO.from_unix/2`, `System.convert_time_unit/3`, and `NaiveDateTime.*` were driven with hostile values from this module's entry points and returned error tuples / clean `ArgumentError`s as expected. I did not separately review their internals; a latent bug in one of those would surface in its own file's review, not here.
- Reproductions ran on `elixir 1.21.0-dev` / OTP 29 built from the tree under review; behaviour on other versions was not checked.

## Did I verify this?

- `diff`/`add` non-positive integer unit → executed; rejected by `System.convert_time_unit` with a clean `ArgumentError`. **Verified (executed).**
- `from_iso8601` on garbage / empty / oversized / invalid-UTF-8 input → executed; all `{:error, _}`, no crash. **Verified (executed).**
- Large-integer `from_gregorian_seconds`/`from_unix` bounds → executed; bignum result or `{:error, :invalid_unix_time}`. **Verified (executed).**
- String in `calendar` field does not dispatch to code → executed; raises on `apply`, no invocation. **Verified (executed).**
- All raise sites are inert `ArgumentError` messages → read every raise site in the full 2160-line file. **Verified (read).**
- Neighbour module internals (parser/`System`/`NaiveDateTime` implementation correctness) → **not independently audited**; only their observable returns under hostile input were confirmed.

### Dropped in re-review

None. Every first-pass dismissal is preserved; the four that carried an "outside this file" hedge (`calendar` dispatch, `from_iso8601`, large-integer consumption, `diff`/`add` unit guard) were upgraded from asserted/predicted downstream behaviour to `verified` with cited executed evidence. No dismissal was moved toward acceptance without executed proof, and no prior finding was silently removed.
