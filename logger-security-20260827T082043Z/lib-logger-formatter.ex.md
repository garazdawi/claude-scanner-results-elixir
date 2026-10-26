# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/formatter.ex
<!-- sha256: 945f43206536e9390caee4bd2e9b12b2f100211aab8c6bb7e2fe2f21d08e34a1 -->

_Analysed on 2026-08-27T08:38:33Z_

Analysis complete. I traced every security-relevant sink and reproduced the reachable ones against Elixir 1.21.0-dev (matching the source under review).

---

## Summary

The formatter is largely a safe transformer of trusted log-event structure; the only sink of note is `compile/1` creating unvalidated atoms from format-pattern fragments (weak trust boundary), while message/metadata content is passed through verbatim by design.

## Security

#### `CSCAN-formatter-compile-unsafe-atom`
- **Finding ID:** `CSCAN-formatter-compile-unsafe-atom`
- **Severity:** Low
- **Confidence:** high
- **Location:** `formatter.ex:444`, `compile/1` (`def compile(str) when is_binary(str)`), validated only afterwards in `compile_code/1` at `formatter.ex:455-459`.
- **Class:** Atom creation (CWE-770)
- **Impact:** Every distinct `$[a-z]+` fragment in a format pattern passed to the public `Logger.Formatter.compile/1` (and thus `Logger.Formatter.new(format: ...)`) is converted to an atom **before** it is validated against `@valid_patterns`. The atom is created and permanently retained even when the fragment is an *invalid* pattern that makes the call raise `ArgumentError` — so unbounded distinct fragments monotonically grow the never-GC'd atom table until the VM aborts. A single fragment longer than 255 characters raises `SystemLimitError`. This only bites if an application feeds attacker/dependency-influenced data as a *log format pattern* — an unusual but documented use of `compile/1` ("can also be used by those interested in building custom formatters"); log format patterns are normally operator/build configuration, which keeps this Low.
- **Evidence:** `verified`.
  - Atom created before validation, pollutes even on the raising path:
    ```
    $ /opt/elixir/bin/elixir t3.exs
    new atoms from 500 invalid fragments: 512
    ```
    (500 distinct `$zzq…` fragments, each of which makes `compile/1` raise `ArgumentError`, still added 512 permanent atoms.)
  - Over-long fragment crashes:
    ```
    $ /opt/elixir/bin/elixir t2.exs
    long-fragment raised: SystemLimitError
    bogus: {ArgumentError, "$bogus is an invalid format pattern"}
    ```
  - `String.to_unsafe_atom/1` is `binary_to_atom` with no existence check: `String.to_unsafe_atom(String.duplicate("a",300))` → `SystemLimitError`.
- **Suggested fix:** Validate the fragment as a *string* before it becomes an atom, e.g. match `code` against the known pattern names (`code in ~w(time date message level node metadata levelpad)`) and only then map to the atom, or use `String.to_existing_atom/1` inside a `try` and treat `ArgumentError` as "invalid pattern". Either removes both the table-growth and the `SystemLimitError` sink while preserving the existing "invalid format pattern" error for unknown fragments.

## Risks / caveats

- **Log injection / terminal-escape smuggling (by design, not flagged):** neither `colorize/4` (`formatter.ex:243-277`, emits raw `IO.ANSI` sequences) nor `prune/1` (`formatter.ex:358-365`, replaces only invalid code points) strips CR/LF, C0 controls, or CSI/ANSI escapes from attacker-influenced message or metadata content. This is CWE-117 territory but is long-standing, documented Logger behavior (multi-line messages are legitimate; sanitizing is the sink's responsibility), so I did not raise it as a defect in this file.
- **`format_report` inspection cost is operator-gated:** `format_report/2` (`formatter.ex:315-321`) runs `Kernel.inspect/2` with `translator_inspect_opts` *before* truncation. Verified the default is `[]` (inspect `limit: 50`) and `truncate: 8096`, so attacker-influenced crash reports are bounded by default; a DoS here requires an operator to set `limit: :infinity`, i.e. a misconfiguration footgun, not a reachable defect.
- I could not exercise `truncate_n`/`scan_inspect` end-to-end from this file alone (they live in `utils.ex`, out of scope); I confirmed the `wrapped_truncate` binary path (`formatter.ex:327`) cannot raise the `ArgumentError` that `truncate_n` throws for non-chardata, because binaries are always valid chardata.
- Reproductions used the installed `/opt/elixir` (1.21.0-dev); I confirmed its `Logger.Formatter.compile/1` behavior and source (`String.to_unsafe_atom`, validate-after) match the file under review.

### Ruled out
- **`apply(mod, fun, [...])` in `format/5` (`formatter.ex:496`)** — the `{mod, fun}` originates from `options[:format]`/`compile({mod,fun})`, i.e. operator configuration (a documented feature), not attacker-forwarded data.
- **`callback.(data)` / `callback.(data, opts)` report_cb dispatch (`formatter.ex:293-297`)** — `report_cb` is a function set by the (trusted) logging call site per the OTP contract; attacker controls report *data*, not the callback. The `callback != (&:logger.format_otp_report/1)` guard prevents self-recursion into the default.
- **Malformed `:domain` metadata → `ArgumentError`** (`metadata(:domain, ...)`, `formatter.ex:558-560`): reproduced a crash with `domain: [:a, "b"]`, but `:domain` is infra-set and typed `[atom()]`; triggering it requires the caller to violate that contract, and attacker-logged *values* don't reach it.
- **`system_time_to_date_time_ms/2` with an out-of-range integer** — verified no crash (`999_999_999_999_999_999_999` → a nonsense year, no exception), and `meta.time` is trusted infra metadata guarded by `is_integer and >= 0` at `formatter.ex:213`.
- **`Regex.split` pattern in `compile/1` (`formatter.ex:440`)** — `\$[a-z]+` is linear with no nested quantifiers; no ReDoS.
- **`String.Chars` protocol dispatch in `metadata/2` (`formatter.ex:577-579`)** — the impl is selected by the value's type, chosen by the trusted call site, not by attacker-controlled data content.

