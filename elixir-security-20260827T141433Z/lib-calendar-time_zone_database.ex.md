# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/time_zone_database.ex
<!-- sha256: eef5f454bc8ed5938bebfd12188fde06fe50e82eafc44fdaded28b09fea211c9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:11:23Z_

Verified. The behavior is exactly as the source reads: `"Etc/UTC"` returns a static map; **every** other input — including a 100,000-char string, a path-traversal string, a bidi-control string, and a novel identifier — returns the static `{:error, :utc_only_time_zone_database}` tuple. Critically, the novel timezone string did **not** become an atom (`:no_atom_not_created`), and large input returns in O(1). The +247 atom delta is Elixir boot/compile noise, not from the DB calls (proven by the `to_existing_atom` check).

# Security review — `calendar/time_zone_database.ex`

## Summary
This file defines a time-zone-database *behaviour* (types + two callback declarations, no executable code) plus a trivial `UTCOnlyTimeZoneDatabase` "null object" implementation whose functions do nothing but pattern-match a timezone string against the literal `"Etc/UTC"` and return a static map or error tuple; it contains no security-relevant sinks.

## Security

No findings.

There is no path in this file that compiles/evaluates code, creates atoms from input, spawns processes, touches the filesystem or network, mutates shared state, decodes/deserialises bytes, dispatches on externally-chosen modules, or performs any input-cost-dependent work. Both `UTCOnlyTimeZoneDatabase` functions are total, constant-return, and input-independent apart from an exact binary equality test against a hard-coded literal.

### Ruled out

- **Atom creation / atom-table exhaustion (CWE-770).** The timezone argument is compared with `==`-style binary pattern matching against the literal `"Etc/UTC"`; it is never passed to `String.to_atom`, `Module.concat`, `binary_to_atom`, or `apply`. Basis: *verified* — feeding `"atom_injection_attempt_xyz_9931"` through both public callbacks left it non-existent as an atom (`String.to_existing_atom` raised `ArgumentError` → `:no_atom_not_created`), and no per-input atom growth occurred.
- **Resource consumption / algorithmic DoS.** A 100,000-byte timezone string returned `{:error, …}` immediately; matching a variable-length binary against a fixed literal prefix is O(1), with no loops, regex, or recursion over the input. Basis: *verified* (test output, `len=100000` → static error).
- **Behaviour / protocol dispatch.** This file *declares* the `Calendar.TimeZoneDatabase` behaviour but performs no dynamic dispatch on any externally-supplied module name; `UTCOnlyTimeZoneDatabase` calls nothing. A hostile *implementation* installed by an operator is out of scope (operator already has code execution). Basis: *asserted* (no `apply`/module-name call sites exist in the file — full file read).
- **Validation correctness.** `UTCOnlyTimeZoneDatabase` is a null-object, not a security gate: it fails closed by returning `:error` for every non-UTC zone, exactly as documented. Its return value is not a "this is safe" predicate. Basis: *verified* (every non-UTC input returned `:error`).
- **Unicode / text confusables.** The only string operation is exact byte-for-byte equality against a literal; there is no normalization, case-folding, or identifier handling here, so no confusable/NFC/NFKC divergence is possible. A bidi character in the input is inert (returned as a plain `:error`). Basis: *verified* (`\u202E`-prefixed input → static `:error`, no special handling).
- **Dead / unreachable defensive code.** Each function's two clauses (the `"Etc/UTC"` clause and the catch-all `_` clause) are both reachable and were both exercised. Basis: *verified* (test drove the UTC and non-UTC branches).
- **Round-trip / deserialisation / command / file / network / crypto / compile-time execution.** None of these sink shapes appear anywhere in the file. Basis: *asserted* (complete 103-line read; no corresponding call sites).

## Risks / caveats
- Scope is strictly this file. The security-interesting behavior of a real, non-UTC time zone database (IANA tzdata parsing, DST transition math) lives in third-party implementations of this behaviour (e.g. `tzdata`, `tz`) and in callers in `Calendar`/`DateTime`; those are out of scope here and were not reviewed.
- The observed +247 atom-table delta is attributable to Elixir/compiler boot and loading the test script's own modules (`IO`, `String`, comprehension), not to the timezone inputs — established by the `to_existing_atom` negative check on a novel input string. I did not otherwise isolate boot atoms, but the negative check is dispositive for the input-driven concern.
- No behavioral claim in this report is `asserted`-only where it could have been run: the atom, resource, validation, unicode, and dead-code rulings were reproduced against the source-built `elixir` 1.21.0-dev. The remaining `asserted` rulings are pure absence-of-sink observations from the full file read, for which there is no code path to execute.

