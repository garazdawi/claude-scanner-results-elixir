# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/uri.ex
<!-- sha256: 12ffc79551caad9484b8c98c885faf3ab5db1d84bed9fe51307071cd274f8c91 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/uri.ex
<!-- sha256: 12ffc79551caad9484b8c98c885faf3ab5db1d84bed9fe51307071cd274f8c91 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (hedge-resolution pass over the first-pass report)_

## Summary

The `URI` module is largely sound. Its one substantive defect is unchanged from the first pass: `parse/1`/`merge/2` convert an unbounded run of port digits to an integer via `String.to_integer/1`, giving superlinear CPU and a hard `SystemLimitError` crash on a crafted URL — from a documented public API whose `@spec` promises totality.

The first pass carried two hedges scoped as "outside this file," both concerning the `new/1`/`new!/1` crash path that terminates inside Erlang's `:uri_string`. I resolved both by **execution**, capturing exact stacktraces (Erlang/OTP 29, `ELIXIR_ERL_OPTIONS="+fnu"`):

- `parse/1` and `merge/2` crash at `URI.split_authority/1` → `uri.ex:913` (this file). **Verified.**
- `new/1` and `new!/1` crash at `:uri_string.get_port/1` → `uri_string.erl:1814`, reached via `:uri_string.parse_reg_name/2` (`:1463`). **Verified** — the crash site is stdlib, not this file, so the `uri.ex:913` fix does *not* cover them.

Because the crash site for `new/1`/`new!/1` is genuinely in stdlib but the **public entry point and the missing length guard at the call boundary live in this file** (`uri.ex:648`, `:692`), I have **upgraded** the previously out-of-scope caveat into a distinct low-confidence finding rather than leaving it as a scope hedge (see `CSCAN-uri-new-uri-string-port-dos`).

---

**Security**

#### `CSCAN-uri-port-to-integer-dos`
- **Finding ID** — `CSCAN-uri-port-to-integer-dos`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/uri.ex:913`, `split_authority/1` (reached from `parse/1` at `:874`, and from `merge/2` at `:991-993`)
- **Class** — Resource consumption (CWE-770: allocation without limits / uncontrolled `binary_to_integer`)
- **Issue** — `split_authority/1` captures the port with the regex group `(:(\d*))?` and then does `port = if nilify(port), do: String.to_integer(port)`. The digit run is unbounded, so `String.to_integer/1` (`:erlang.binary_to_integer/1`) builds an arbitrarily large bignum from attacker-controlled text. Cost is superlinear in the number of digits; beyond ~1.3 million digits the VM raises `SystemLimitError`. `parse/1` is documented as non-validating and its `@spec parse(t | binary) :: t` promises a struct for every binary — it never advertises that it can raise, so callers treat it as total.
- **Impact / threat** — A single crafted URL of the form `//host:999…9` reliably makes `URI.parse/1` and `URI.merge/2` raise `SystemLimitError` instead of returning; at hundreds of KB it burns superlinear CPU first. Trust boundary: any code that feeds an externally supplied URL (redirect `Location`, user link, feed item, webhook target, SSRF allow-list check) to `URI.parse` — the near-universal assumption is that it cannot throw. Result: an unexpected exception that can fail a request handler, poison a background job, or crash a URL-parsing `GenServer`, plus attacker-controlled CPU spend. Bounded by BEAM process isolation and the ~1.3 MB input for the hard crash, hence Low.
- **Evidence** — `verified` (re-confirmed this pass by execution).
  - Crash + exact location: `URI.parse("//a:" <> String.duplicate("9", 2_000_000))` →\
    `SystemLimitError`, stacktrace top frames: `{:erlang, :binary_to_integer, [<2M digits>], [error_info: …]}` → `{URI, :split_authority, 1, [file: ~c"lib/uri.ex", line: 913]}` → `{URI, :parse, 1, [file: ~c"lib/uri.ex", line: 874]}`.
  - `merge/2` same input: identical top frames ending at `uri.ex:913` → `uri.ex:874` (via `merge(parse(base), parse(rel))` at `:991-993`).
  - First-pass measurements retained: threshold `digits=1_200_000 -> ok`, `1_400_000 -> crash`; superlinear CPU below threshold (`200000 → 10.9 ms`, `400000 → 26.3 ms`, `800000 → 75.2 ms`).
- **Suggested fix** — Bound the port length before converting: only call `String.to_integer/1` when the captured digit string is of sane length (a valid RFC 3986 port is ≤ 5 digits; even `byte_size(port) <= 10` removes the pathological bignum). Equivalently, tighten the capture (e.g. `\d{1,10}`) so an over-long port is not treated numerically. Since `parse/1` is intentionally total, treat an over-long run as "not a valid port."

#### `CSCAN-uri-new-uri-string-port-dos` — Upgraded from hedged assumption in first pass
- **Finding ID** — `CSCAN-uri-new-uri-string-port-dos`
- **Severity** — Low
- **Confidence** — low
- **Location** — `lib/uri.ex:648` (`new/1`) and `lib/uri.ex:692` (`new!/1`), which call `:uri_string.parse/1`; crash executes at `uri_string.erl:1814` (`:uri_string.get_port/1`), stdlib.
- **Class** — Resource consumption (CWE-770), reachable-but-root-cause-in-dependency
- **Issue** — `URI.new/1` and `URI.new!/1` are public functions defined in this file that forward the entire attacker-controlled binary to `:uri_string.parse/1` with no length pre-check. `:uri_string` performs the same unguarded `:erlang.binary_to_integer/1` on the port digit run inside `get_port/1`, so the identical long-port input crashes these entry points too. The first pass dismissed this as "out of this file's scope" because the failing `binary_to_integer` is in stdlib; I am recording it as a low-confidence finding because the exposed API surface and the absent guard-at-the-boundary are in *this* file, and — importantly — the fix proposed for `CSCAN-uri-port-to-integer-dos` (guarding `split_authority/1`) does **not** protect `new/1`/`new!/1`, which never route through `split_authority/1`.
- **Impact / threat** — Same DoS profile as the primary finding but via the *validating* constructors that callers reach for precisely when they want a safe/total parse: `URI.new("//a:" <> big)` raises `SystemLimitError` rather than returning `{:error, _}`, and `URI.new!/1` raises it rather than its documented `ArgumentError`. A caller that switched from `parse/1` to `new/1` to "do it safely" is still crashable. Low because the root cause is in OTP stdlib (any real remediation is upstream) and BEAM process isolation bounds blast radius.
- **Evidence** — `verified` reachability, `low` confidence on the right remediation owner.
  - `URI.new("//a:" <> String.duplicate("9", 2_000_000))` → `SystemLimitError`, top frames `{:erlang, :binary_to_integer, …}` → `{:uri_string, :get_port, 1, [file: ~c"uri_string.erl", line: 1814]}` → `{:uri_string, :parse_reg_name, 2, [file: ~c"uri_string.erl", line: 1463]}`. `new!/1` identical.
  - Contrast confirms scope split: `parse/1`/`merge/2` crash at `uri.ex:913`; `new/1`/`new!/1` crash at `uri_string.erl:1814`. So the two paths need two independent fixes.
- **Suggested fix** — In `new/1`/`new!/1`, apply a defensive length/shape check on the input (or specifically reject over-long port runs) before delegating to `:uri_string.parse/1`; and/or report the unbounded `get_port/1` to OTP as the upstream fix. At minimum, document that `new!/1` can raise `SystemLimitError`, not only `ArgumentError`.

### Ruled out
- **ReDoS in the `parse/1` and `split_authority/1` regexes** — `verified`. Both regexes are anchored/linear with no nested quantifiers; adversarial inputs stayed linear: 500 KB `no-@` authority = 6.6 ms, 250 K `a@` repeats = 0.65 ms, 500 KB userinfo = 0.69 ms, 1 MB path/query = 0.68 ms. No catastrophic backtracking.
- **Quadratic blowup in `encode_*`/`decode_*` binary accumulators** — `verified` linear. `<<acc::binary, …>>` accumulators are BEAM-optimized: decode 250 K/500 K/1 M `%20` = 1.1/2.3/4.5 ms; encode = 4.4/8.7/17.3 ms (both scale linearly).
- **Crash / path traversal in `merge/2` dot-segment handling** — `verified`. Fuzzed relative parts (`../../../../../etc`, `..`, `.`, `//other@x`, `g:h`, `////`, `../%2e%2e/`, empty, `/`) produced no crash; `..` correctly stops at root. URL-path normalization only — no filesystem path derived here, so no zip-slip/traversal sink.
- **Deprecated `decode_query/3` struct path (`String.to_unsafe_atom("Dict")`, `lib/uri.ex:255`)** — `verified` not an atom-exhaustion or code-execution sink. The atom is the constant `"Dict"` (no external bytes → atom), and the path just raises `UndefinedFunctionError`. Reaching it requires the *caller* to pass a struct as the accumulator, not attacker-controlled data.
- **Atom creation from URL content** — `verified` by reading every conversion: the only atomization is the constant `"Dict"` above; `decode_query`/`query_decoder` keys and values are percent-decoded binaries, never atoms. No `String.to_atom`/`Module.concat` on external input.
- **`parse/1` host/userinfo permissiveness (double-`@`, empty host, backslash) as a defect in this file** — Not raised as a finding. Verified the permissive behavior (`parse("http://user@host@evil.com/")` → `host: "evil.com"`, `userinfo: "user@host"`; `parse("http://user@:80")` → `host: nil`), but this is the documented, intentional contract of `parse/1`, and the validating alternative `new/1` rejects the same inputs (`{:error, ":"}`). The risk lives in *consumer* code that trust-checks `parse` output. See Risks below.

---

**Risks / caveats**

- **`parse/1` is deliberately non-validating and must not back a trust decision.** It returns a `host`/`userinfo` split that differs from a strict RFC-3986 or WHATWG parser (verified: last-`@` wins for userinfo; empty host becomes `nil` while `userinfo` is retained; backslashes and `]` pass through). Any host allow-listing, scheme check, or SSRF guard built on `URI.parse` output can be bypassed. Callers needing validation must use `URI.new/1` (verified to reject these). Correct-as-documented, so residual-risk, not a finding.
- **The Erlang side is confirmed *not* bounded (previously an unverified caveat).** The first pass wrote "a complete fix should confirm the Erlang side is bounded too, but that is outside this file's scope." Resolved by execution: `:uri_string.get_port/1` (`uri_string.erl:1814`) performs the same unguarded `binary_to_integer`, so `new/1`/`new!/1` crash identically. This is now the separate finding `CSCAN-uri-new-uri-string-port-dos` rather than a hedge. Any real remediation for those two entry points is upstream in OTP or a guard added in this file's `new/1`/`new!/1`.
- **`String.Chars.URI.to_string/1` performs no escaping** and reproduces struct fields verbatim (by design — fields are contractually pre-encoded). No `parse → to_string` round-trip that changes meaning was found (verified IPv6 re-bracketing and the double-`@` case round-trip consistently). A caller that stuffs unencoded control characters (e.g. CRLF) directly into fields and serializes could enable downstream header/log injection, but that requires violating the documented field-encoding contract, so out of scope here.
- Benchmarks are single-run wall-clock on this machine (Erlang/OTP 29, JIT); absolute milliseconds vary, but scaling relationships and the crash threshold reproduce deterministically.

---

### Did I verify this
- **Hedge 1 — "`new/1`/`new!/1` crash inside Erlang's `:uri_string`, out of this file's scope."** Verified by execution. Captured stacktraces: `new/1` and `new!/1` terminate at `{:uri_string, :get_port, 1, [file: ~c"uri_string.erl", line: 1814]}` via `parse_reg_name/2` (`:1463`), *not* at `uri.ex:913`. The scope claim is true and now execution-cited, not assumed.
- **Hedge 2 — "a complete fix should confirm the Erlang side is bounded too."** Verified by execution: the Erlang side is unbounded and crashes with the same `SystemLimitError`. Upgraded into finding `CSCAN-uri-new-uri-string-port-dos` (low confidence) because the vulnerable public entry points (`new/1`/`new!/1`) are defined in this file and the primary fix does not cover them.
- **Primary finding (`uri.ex:913`).** Re-confirmed this pass by execution: `parse/1` and `merge/2` crash at `URI.split_authority/1` → `uri.ex:913` → `uri.ex:874`. High confidence retained.
- **Neighbour reads vs. axis-tries.** The relevant neighbour here is the stdlib `:uri_string` module (not one of the compiler `src` modules in the standing context, which do not call into `uri.ex`). I resolved the two hedges by *trying the axis* — pushing a 2M-digit-port URL through `parse/1`, `merge/2`, `new/1`, and `new!/1` and comparing crash sites — rather than by reading, exactly because reading would not tell me which entry points diverge. That divergence (`uri.ex:913` vs `uri_string.erl:1814`) is the evidence for splitting the finding.

### Dropped in re-review
- None. All first-pass findings and ruled-out entries are preserved. The two first-pass "outside this file" scope caveats were resolved: hedge 1 is now a verified (execution-cited) scope statement; hedge 2 is upgraded to the low-confidence finding `CSCAN-uri-new-uri-string-port-dos`.
