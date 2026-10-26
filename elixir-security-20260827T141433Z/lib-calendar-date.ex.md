# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/date.ex
<!-- sha256: 53ffb5eb1e9fe21ea197ec60d0d9f061e5ef7486649f9ce4116123d07bfe3378 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/date.ex
<!-- sha256: 53ffb5eb1e9fe21ea197ec60d0d9f061e5ef7486649f9ce4116123d07bfe3378 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

The first-pass reviewer flagged one hedge class (**"outside this file"**), concentrated in the Risks/caveats note that the actual byte-level parser (`Calendar.ISO.parse_date/1`) was "outside this file's scope" and "not audited." I resolved that hedge by reading the neighbour module (`lib/calendar/iso.ex`, the callgraph successor of `from_iso8601/2`). The parser is now verified, and the caveat is downgraded from hedged to verified below.

---

# Security review — `lib/elixir/lib/calendar/date.ex`

## Summary
The `Date` standard-library module is a pure date-manipulation module whose one untrusted-input entry point (`from_iso8601/2`) delegates all parsing to `Calendar.ISO.parse_date/1` and rejects hostile input cleanly. Having now read the delegated parser, I confirm the forward-and-handle chain is bounded and atom-free end to end. No security-meaningful defect is reachable through the documented public API.

## Security

`No findings`

I inventoried every documented public function against the Phase 1 sink classes. The only entry point that consumes attacker-shaped data is `from_iso8601/2` (and its `!` variant); every other function takes `Date`/`Calendar.date()` structs or integers that must satisfy the typespec. Verified behaviors:

- **Untrusted-string parsing (`from_iso8601/2`)** — delegates to `Calendar.ISO.parse_date/1` and, on the string content, only ever produces integers placed into a fixed `Calendar.ISO` struct (date.ex:361–362). *Verified:* `Date.from_iso8601(<<0xff,0xfe,"abc">>)` → `{:error, :invalid_format}`; `Date.from_iso8601(String.duplicate("9",1000)<>"-01-01")` → `{:error, :invalid_format}`.
- **No atom creation** — no `String.to_atom`/`Module.concat`/`binary_to_atom` in this file *or in the delegated parser* (see Did-I-verify below). *Verified:* atom-count delta over 20,000 distinct hostile `from_iso8601` inputs was **0**.
- **`shift/2` duration validation** — `validate_duration_unit!/1` rejects time units, unknown units, and non-integer values *before* `struct!(Duration, unit_pairs)` (date.ex:826–846), so a caller keyword list cannot smuggle a `__struct__` key. *Verified:* `hour: 2`, `bogus: 2`, `month: 1.5`, and `__struct__: Evil` all raise `ArgumentError` and never reach `struct!`.
- **No resource sink in-file** — `range/2,3` build the `Date.Range` struct in O(1); `from_gregorian_days/2` is constant-time; range *enumeration* cost lives in the separate `Date.Range` Enumerable impl.
- **No** code/command execution, file/path handling, deserialization, crypto comparison, or writes to shared mutable state.

### Ruled out

- **Byte-level ISO-8601 parser (`Calendar.ISO.parse_date/1`) — Untrusted-input parsing / atom-exhaustion / unbounded-CPU.** **Upgraded from hedged assumption in first pass** (the first pass wrote "did not audit that parser itself"), now resolved to **ruled out — verified by reading the neighbour**. `parse_date/1` → `parse_date/2` → `parse_date_guarded` → `do_parse_date` (iso.ex:403–460). `do_parse_date` matches the source against a **compile-time fixed-width binary pattern** of exactly 8 bytes for the date (`<<y1,y2,y3,y4,@ext_date_sep,m1,m2,@ext_date_sep,d1,d2>>`, iso.ex:227–241), guarded so every byte is an ASCII digit. Consequences relevant to the sink classes:
  - **Year is structurally capped at 4 digits** — `read_date` computes `y1*1000 + … + y4` from four bytes; there is no variable-length digit run, so the "1000-digit year" DoS input cannot produce a large integer or unbounded work — it simply fails the fixed-width match.
  - **No `rest::binary`** in the date clauses ⇒ trailing junk is rejected, not silently accepted.
  - **No atom construction** anywhere in the parse path; the fall-through `do_parse_date(_,_,_)` returns `{:error, :invalid_format}` and `parse_formatted_date` returns `{:error, :invalid_date}` via `valid_date?` (iso.ex:448–459). This is the source-level explanation of the measured 0-atom / clean-reject behavior.
  *Basis: verified* — read iso.ex:200–241, 403–460.
- **Behaviour dispatch on the `calendar` module** (`calendar.date_to_string(...)`, `calendar.shift_date(...)`, etc.) — the dispatched module always comes from a caller-supplied struct field or an explicit calendar argument, never from data parsed out of untrusted input. In `from_iso8601/2` the parsed date is hardcoded to `%Date{... }` (implicitly `Calendar.ISO`) before `convert/2`, and the target `calendar` argument is independent of the untrusted string (date.ex:360–362). Reaching an attacker-chosen module requires the caller to place an untrusted atom into `%Date{calendar: ...}`, itself a `Calendar.calendar()` contract violation and caller-side atom-creation bug. *Basis: verified* — date.ex:361–362, 468–469.
- **Round-trip asymmetry between `to_iso8601/2` and `from_iso8601/2`** for `|year| > 9999` — documented ("year parsed … is limited to four digits", date.ex:345) and confirmed to be a *reject*, not a silent value-confusion: the 4-digit fixed-width match in iso.ex:230–234 means a 5-digit year string cannot match at all. *Basis: verified* — `to_iso8601(Date.new!(12345,1,1))` → `"12345-01-01"`, and `from_iso8601("12345-01-01")` → `{:error, :invalid_format}`; 4-digit and negative-4-digit years round-trip exactly.
- **`from_erl/2` with non-integer/wrong-arity tuples** — only reachable by violating the `:calendar.date()` typespec; invalid-but-well-typed tuples flow through `new/4`'s `calendar.valid_date?` guard (date.ex:287, 468–469) and return `{:error, :invalid_date}`. *Basis: verified* — the `new/4` validation path in this file (date.ex:286–292).
- **`Inspect` implementation emitting `Date.new!(...)` code-like text** — all interpolated components are integers via `Integer.to_string/1` (date.ex:1183–1189), so there is no injection even if inspect output were evaluated (which would itself be out of scope). *Basis: verified* — date.ex:1183–1189.

## Risks / caveats
- **Parser now audited.** The prior report's caveat that the byte-level parser was outside scope and unaudited has been resolved: I read `Calendar.ISO.parse_date/1` and its supporting `do_parse_date`/`parse_formatted_date` clauses (iso.ex:403–460) and the fixed-width match macros (iso.ex:227–241). It is bounded (fixed 8-byte date match, 4-digit year cap), atom-free, and rejects malformed/oversized/trailing input with `{:error, :invalid_format | :invalid_date}`. No defect found in the delegated path either.
- Findings were judged against the documented public API only, per scope. Callers that construct `%Date{}` structs directly with an untrusted `:calendar` atom (contrary to the module's explicit guidance to use the provided constructors, date.ex:30–32) can steer the `calendar.*` dispatches; that is a caller-side contract/atom-creation issue, not a defect in this file. This is a scope boundary, not an unverified assumption about another file.
- Reproductions ran on `Elixir 1.21.0-dev` / OTP 29 built from the source tree under review.

## Did I verify this?
- **Delegated parser (`Calendar.ISO.parse_date`)** — YES, read directly (iso.ex:403–460, 227–241). Fixed-width match, 4-digit year cap, no atoms, no `rest::binary` trailing acceptance, clean error tuples. Fully resolves the "outside this file" hedge.
- **In-file forwarding / result handling of `from_iso8601/2`** — YES, read (date.ex:360–364) and executed (hostile-input reproductions, 0-atom delta over 20,000 inputs).
- **`shift/2` duration validation, `from_erl/2`, `Inspect`, `range/*`** — YES, read in-file and (for shift/inspect/round-trip) executed.
- **The `calendar.*` behaviour-dispatch ruled-out** — YES for the claim that no *documented API* routes untrusted data into the dispatched module (read date.ex:361–362, 468–469); the residual (caller directly builds a hostile struct) is an out-of-scope caller contract, stated as such, not an unverified cross-file assumption.

## Dropped in re-review
None. All prior conclusions are preserved; the single "parser outside scope / not audited" hedge in the first pass has been **upgraded to a verified ruled-out entry** rather than dropped, on the strength of reading `lib/calendar/iso.ex`.
