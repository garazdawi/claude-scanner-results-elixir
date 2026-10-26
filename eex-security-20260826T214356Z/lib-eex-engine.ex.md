# /Users/lukas/code/claude-scanner/elixir/lib/eex/lib/eex/engine.ex
<!-- sha256: 863d0beb23bb19e97c3b2e0cec8cf374223ba2a146aa139328e23319b2c4c66e -->

_Analysed on 2026-08-26T21:54:30Z_

Both runtime-relevant sinks are verified clean: 500 missing-assign evals created **0 atoms**, and the `argN` compile-time atoms are deterministic and reused (0 new atoms on a second same-size template). I have enough to write the report.

## Summary

`EEx.Engine` is a set of compile-time AST builders plus one runtime assign-fetcher; every untrusted-byte path into it is the documented "template = code" (SSTI/RCE) contract, and the one genuine runtime channel (`assigns` values/keys via `fetch_assign!/2`) handles hostile input safely — no security findings.

## Security

No findings.

## Risks / caveats

- **Template source is code, by contract.** Every compile-time function here (`init/1`, `handle_text/3`, `handle_expr/3`, `handle_begin/1`, `handle_end/1`, `handle_body/1`, `handle_assign/1`) operates on the parsed template AST. If a caller feeds an attacker-controlled string as the *template* to `EEx.compile_string/2` / `EEx.eval_string/3`, that is arbitrary code execution regardless of anything in this file. This is EEx's documented design, not a defect of this module, and is excluded from scope.
- **No output escaping — by design.** `handle_expr("=", …)` emits `String.Chars.to_string/1` with no HTML/markup escaping (verified: `<script>` passes through verbatim). Contextual escaping is the downstream engine's responsibility (e.g. `Phoenix.HTML.Engine`), not `EEx.Engine`. Do not read this as an XSS bug in this file.
- **`var!(assigns)` is intentionally unhygienic.** `handle_assign/1` rewrites `@foo` into `fetch_assign!(var!(assigns), :foo)`, capturing an `assigns` binding from the caller's scope. This is the documented assign mechanism; the key atom (`:foo`) already exists in the parsed AST, so no atom is created from runtime data.
- **`fetch_assign!/2` on a non-`Access.t()` `assigns`** (e.g. a bare binary) will raise from `Access.fetch/2`, and its missing-key warning path calls `Enum.map(assigns, &elem(&1, 0))`, which assumes an enumerable of tuples/map. Both require the *caller* to violate the `@spec` (`assigns :: Access.t()`); the attacker controls values, not this structural contract, so it is out of scope.
- **Custom engines are trusted.** `check_state!/1` only validates map shape; a custom engine delegating to these defaults with a malformed accumulator gets a clean `raise`, not a security event.

### Ruled out

- **`String.to_unsafe_atom("arg#{vars_count}")` (line 198) — atom-table exhaustion (CWE-770).** Rejected. The input is an *internal monotonic counter*, not externally-derived bytes; the atoms produced are deterministic `arg0..argN` and are **reused across every template**. Verified: a first template with 2001 `<%= %>` expressions grew the atom table by 6051, but a second, same-size template with different content added **0** new atoms (`delta tmpl2(same size,NEW template)=0`). Growth is bounded by the single largest template's expression count, not cumulative; exhausting the 1M-atom table would require one template with ~1M expressions, which is already RCE. This is the same vetted idiom used by `IEx.Pry` for generating unique variable names, and `to_existing_atom/1` cannot substitute because the variable names must be freshly created.
- **`fetch_assign!/2` (line 140) — atom creation / crash / DoS from hostile `assigns`.** Rejected. Runtime channel with attacker-controlled values (and possibly keys). Verified: 500 evaluations with a missing assign created **0 atoms**, returned `nil` (rendered as `""`), and did not crash; the key is always a compile-time literal atom, never derived from runtime data.
- **`IO.warn` with attacker-controlled assign keys (line 148) — log injection (CWE-117).** Rejected. Keys are emitted via `inspect(keys)`, which quotes and escapes control/terminal bytes (verified: `"evil<script>"` appeared quoted inside the warning), and the message is a developer-facing compiler/runtime warning, not an attacker-facing sink.
- **`String.Chars.to_string/1` protocol dispatch (line 202) — injection via a malicious `String.Chars` impl.** Rejected. Dispatch resolves on the runtime value's *type*; an attacker supplying a value cannot register a new protocol implementation for an existing type, and missing-escaping is by design (see caveats).
- **`check_state!/1` raise (lines 225-228) — dead defensive code.** Rejected as a finding: it is reachable (a custom engine delegating with a non-conforming state map triggers it) and behaves correctly; not dead, not a security defect.

