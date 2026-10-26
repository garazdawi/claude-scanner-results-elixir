# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/time.ex
<!-- sha256: d67146cf725623177ac6bcf8d13ea5e5dec42a874321a67f198c51b936814c78 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/calendar/time.ex
<!-- sha256: d67146cf725623177ac6bcf8d13ea5e5dec42a874321a67f198c51b936814c78 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

# Security review — `lib/elixir/lib/calendar/time.ex`

## Summary
`Time` is a pure, side-effect-free calendar/struct module. Its public API performs only bounded integer/rational arithmetic and polymorphic dispatch on the caller-supplied calendar module. The one untrusted-string entry (`from_iso8601/2`) delegates to `Calendar.ISO.parse_time/1`; I read that delegate this pass and executed the previously-unexamined resource-exhaustion axes against it. No security-relevant sink is reachable through this file, and the delegated parse path is linear and bounded. No findings.

## Security

`No findings`

The file was checked against every Phase-1 sink class; none are present:

- **No code execution / atom creation / command exec / file ops / deserialization / crypto / regex / templating / shared mutable state.** A grep for the full dangerous-sink name set (`to_atom`, `Module.concat`, `Code.*`, `apply`, `System.cmd`/`shell`, `File.*`, `Path.*`, `binary_to_term`, `Regex`/`~r`, `Base`, `URI`, `:crypto`, `:ets`, `persistent_term`, `put_env`, `Process.put`, `EEx`, `:erlang.*`) returns zero matches.
- The only string-typed public entry, `from_iso8601/2` (line 294), delegates parsing to `Calendar.ISO.parse_time/1` and only wraps the result in a struct + `convert/2`. This file adds no parsing/decoding logic of its own. (Delegate now read — see Ruled out.)
- All integer-typed entries (`new/5`, `add/3`, `from_seconds_after_midnight/3`, `diff/3`) are guarded by `is_integer/1` and reduce to cyclic/modular arithmetic, so cost is bounded regardless of input magnitude.

### Ruled out

- **Resource exhaustion via `from_iso8601/2`.** `verified` — Axes examined: fractional-digit count; non-numeric/garbage prefix length; trailing-content length (offset-parse path); overall input size. | Not examined: none.
  I read the full delegate chain in `calendar/iso.ex`: `parse_time/1` (line 322) → `do_parse_time` (352–366, hour/minute/second matched by **fixed-width** patterns `match_ext_time`/`match_basic_time`, so leading fields cannot drive cost) → `parse_formatted_time` (368) → `parse_microsecond/3` (2063–2075, a tail-recursive digit consumer that stops contributing past 6 digits and calls `binary_to_integer` on at most a 6-byte slice `:binary.part(digits, 0, precision)`) → `parse_offset` (2077–2107, fixed-width). Every stage is O(n) single-pass with no quadratic accumulation. Executed evidence (5 MB inputs, OTP 29 / Elixir 1.21.0-dev):
  - `Time.from_iso8601("23:50:07." <> String.duplicate("1", 5_000_000))` → `{:ok, ~T[23:50:07.111111]}` in **9.3 ms** (fractional axis).
  - `Time.from_iso8601(String.duplicate("9", 5_000_000))` → `{:error, :invalid_format}` in **17.3 ms** (garbage-prefix axis).
  - `Time.from_iso8601("23:50:07" <> String.duplicate("Z", 5_000_000))` → `{:error, :invalid_format}` in **15.9 ms** (trailing/offset axis).
  This resolves the first-pass "scope boundary" caveat: the delegate was read and stress-tested, not assumed.
- **Integer/bignum DoS via `add/3` and `from_seconds_after_midnight/3`.** `verified` — Axes examined: magnitude (up to 10^200000); sign (positive and negative); both entry points. | Not examined: none.
  Results are reduced modulo the day (`Integer.mod(seconds, @seconds_per_day)`, line 476; `shift_time_unit` for `add`), so cost does not scale with value. Executed: `Time.add(~T[10:00:00], Integer.pow(10, 200_000), :second)` → hour `19` in **3.1 ms**; `Time.from_seconds_after_midnight(-Integer.pow(10, 100))` → `~T[06:13:20]`. A caller would have to already hold a pathological bignum, which this file never derives from small/untrusted input.
- **Behaviour/protocol dispatch on `calendar`** (`calendar.time_to_string/*`, `time_from_day_fraction/*`, `valid_time?/*`, `shift_time/*`, and the `String.Chars`/`Inspect` impls, lines 254, 1013, 1028). `asserted` — Axes examined: none tried at runtime. | Not examined: whether any in-file path derives the `calendar` atom from a string/decoded input rather than the caller's struct field.
  This is ordinary polymorphic dispatch on `t:Calendar.calendar/0`, an atom module reference read straight off the struct. Directing it at attacker-chosen code requires that module to already be compiled and loaded in the victim VM (i.e. code execution the attacker already has). The dismissal rests on a design invariant, not an executed axis, so it stays `asserted`; a later pass may re-examine whether any caller synthesizes the calendar atom.
- **Error-message data exposure** via `inspect/1` in `new!/216`, `from_iso8601!/326`, `from_erl!/445`. `asserted` — the interpolated values are the caller's own arguments echoed back in an `ArgumentError`; no trust boundary is crossed and no third-party secret is introduced by this module.
- **Validation gates as dead/bypassable defensive code** — `add/3` unit check (577–585) and `validate_duration_unit!/1` (664–692). `asserted` — both are reachable from public entries (`Time.add(t, 1, :bad)`, `Time.shift(t, year: 1)` each raise `ArgumentError`) and are correctness/type gates, not security boundaries; neither is dead.
- **Round-trip integrity** of `to_iso8601` ⇄ `from_iso8601`. `asserted` — microsecond truncation to 6 digits is documented (`parse_microsecond/3` line 2062: "Digits past the sixth are consumed but do not contribute") and symmetric-enough for a time value; there is no signature/cache-key trust decision made on the encoding in this file.

## Risks / caveats
- **Delegated parser now covered for cost, not for correctness of every branch.** I read `Calendar.ISO.parse_time` / `parse_microsecond` / `parse_offset` and confirmed the path is linear and that `time.ex`'s wrapper adds no amplification. I did not exhaustively audit `Calendar.ISO`'s *semantic* correctness (e.g. offset edge cases); any such defect is a finding against `calendar/iso.ex`, not this file, and does not change this file's verdict.
- **Standing-context mismatch (informational, not a hedge).** The folder threat model and Unicode/UAX-31/Trojan-Source obligations supplied with this task describe the Erlang compiler front-end (`lib/elixir/src`). None apply to `time.ex`: it has no identifier, bidi, NFC, or escape-handling logic. Not force-fit.

## Did I verify this?
- **Yes, by reading the neighbour:** `Calendar.ISO.parse_time/1,2`, `do_parse_time`, `parse_formatted_time`, `parse_microsecond`, `parse_offset` in `calendar/iso.ex` (lines 322–379, 2058–2107) — the delegate for `from_iso8601/2`. Confirmed fixed-width field matching and a bounded, single-pass microsecond consumer.
- **Yes, by execution (axes tried, not assumed):** the four resource-exhaustion axes (fractional / garbage-prefix / trailing / magnitude, both signs) above, run under Erlang/OTP 29. Each completed in single-digit-to-teens of milliseconds on 5 MB / 10^200000 inputs.
- **Not verified by execution (remains `asserted`):** the calendar-dispatch dismissal — its axis (whether any in-file path synthesizes the calendar atom from untrusted input) is a static-provenance question I did not attempt to drive at runtime, so it is not cleared.

## Upgraded from hedged assumption in first pass
- The first pass carried the scope-boundary hedge *"I did not audit `Calendar.ISO`; any parser defect there is out of scope."* For the **resource-exhaustion** dimension this is now **verified**, not hedged: the delegate chain was read and the previously-unexamined non-fractional and trailing-content axes were executed, letting the "Resource exhaustion via `from_iso8601/2`" ruled-out entry move to `Not examined: none`. No assumption was upgraded to a finding — the executed evidence supports dismissal, and downgrading a hedge to a verified dismissal is the safe direction.

## Dropped in re-review
- None. Every prior ruled-out entry is preserved; two moved from empirically-spot-checked to fully-verified (`Not examined: none`) with the delegate read and the additional axes executed.
