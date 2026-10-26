# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/translator.ex
<!-- sha256: 2d9c610af01347ff6f6a4a557aa2910d18c8ab8bdba6af35210dd28118f5c66e -->

# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/translator.ex
<!-- sha256: 2d9c610af01347ff6f6a4a557aa2910d18c8ab8bdba6af35210dd28118f5c66e -->

_Re-analysed on 2026-08-27_

# Security Review — `logger/translator.ex`

## Summary
`Logger.Translator` renders OTP/crash reports into log chardata, routing most attacker-influenced content through `inspect` (which escapes control characters) with every clause running inside `Logger.Utils.translator/2`'s outer `try/rescue` (verified in `utils.ex:19–55`). Re-review executed the two axes the first pass hedged on: (1) `Exception.format_banner/3` and (2) `Application.format_error/1` (lines 96/208). Both were confirmed by running them to emit **raw** newlines and ANSI escape sequences when the exit reason is an exception (or `{exception, stacktrace}`) whose message is attacker-influenced — so the first-pass "outside this file's responsibility" hedge is upgraded to a single **Low-confidence log-injection finding**. No code-execution, atom-creation, path, command, or deserialization sink is present.

## Security

### Log injection (CWE-117 / CWE-150) via raw exception banners and `Application.format_error/1`
- **Confidence: Low** — **Upgraded from hedged assumption in first pass** ("log injection is real but not this file's defect" / "I did not exhaustively audit `Application.format_error/1`").

The translator renders exception content through two sinks that do **not** escape control characters, verified by execution against OTP 29 / Elixir 1.21.0-dev:

- `Exception.format_banner(:error, exc, stack)` (translator.ex:701, 708, 714, 762) — ran with `%RuntimeError{message: "boom\nFAKE 2026 injected\e[31m"}`, output binary was literally `** (RuntimeError) boom<LF>FAKE 2026 injected<ESC>[31m` — the `\n` and `\e` are **real bytes**, not escaped.
- `Application.format_error/1` (translator.ex:96 for `{{:application_controller, :exit}, …}`, and translator.ex:208 for `{:std_info, …}`) — previously unaudited. Ran it across reason shapes:
  - `{exception, stacktrace}` reason → `"an exception was raised:\n    ** (RuntimeError) boom\nFAKE 2026 injected\e[31m\n        foo.ex:1: Foo.bar/1"` — **raw** LF + ESC reach output.
  - `{:bad_return, …}`, `{:EXIT, {exc, stack}}`, plain atom, plain binary reasons → all `inspect`-escaped (safe).

- **Failure scenario:** An OTP application crashes while a supervised process is handling attacker-controlled data (e.g. `raise "invalid token: #{user_input}"` where `user_input` contains `\n[2026-08-27 00:00:00] [info] admin login OK` or a CSI sequence). The crash produces a `proc_lib`/`crash_report`/`application_controller :exit` report; the translator formats it via `format/3` (→ `format_banner`) or `Application.format_error/1`, emitting the raw newline/ANSI into the log line. A downstream file/terminal sink then shows a forged second log line or executes a terminal escape.

- **Why Low:** This is the documented Elixir exception-rendering format (matching what `Kernel` prints on an uncaught crash), and the folder threat model explicitly assigns CR/LF output policy to the *formatter* layer, noting `prune` deliberately does not strip CR/LF and the default formatter preserves them. The injection root cause is (a) the raising code embedding unsanitized input and (b) the formatter's preserve-CR/LF policy — the translator faithfully delegates. It is recorded as a low-confidence finding rather than a dismissal only because the raw-output mechanism is now executed-confirmed and the file is the component that turns hostile crash content into log text; a fix here (escaping) would diverge from standard Elixir behavior and belongs in the formatter, not the translator.

**Phase 1 sink inventory (and disposition):**

- **Text emission / log-injection** — see finding above. All other emitters — `inspect/1,2` (bounded by `translator_inspect_opts`), `Atom.to_string/1` (87/96/208/228/366), `Integer.to_string/1` (433), `Exception.format_exit` (offender_reason), `Macro.inspect_atom/2` (727), the `#{inspect(...)}` interpolations (235–236) — escape control chars (`inspect` verified to escape `\n`/`\e`; `format_exit` inspects→escapes). `Atom.to_string` on `app`/`state_enter?` is atom→string with no control-char risk.
- **Atom creation** — none. Every `Atom.to_string` is atom→string; no `String.to_atom`, `Module.concat`, or `binary_to_atom`.
- **Code execution / dispatch** — `format_mfa`, `initial_call_to_mfa`, `crash_call` compute/format module-function-arity but never `apply`. No `Code.*`, `eval`, or callable invocation. (The only `apply` in the callgraph is `Logger.Utils.translate/5` at `utils.ex:79`, dispatching to the operator-configured `{mod, fun}` translator list — trusted config, not attacker-shaped.)
- **File / path / command / archive / network / deserialization / crypto / unicode** — none present (grep-verified).
- **Shared mutable state** — reads `Application.get_env(:logger, :translator_inspect_opts)`; writes nothing global. `user_metadata` (report_crash, line 532) comes from the *crashing process's own* dictionary `$logger_metadata$`, not a cross-process channel.
- **Resource consumption** — `inspect` and the `crash_info`/`crash_linked`/`format_stacktrace` recursions are bounded by report size and by default inspect opts.

- **Verification of the containing invariant** — driving the file with malformed/hostile reports, every case resolved to a normal result or an ordinary exception caught by the outer `rescue`; nothing escaped via `throw`/`exit`:
```
statem-bad-reason: OK -> :none
crash-no-dict: RESCUE(FunctionClauseError)  [caught by outer try/rescue]
crash-newline-msg: OK -> {:ok, ["Process ", [...], ...], ...}
genserver-format: OK -> {:ok, [["GenServer ", ...], ...], ...}
sup-weird: OK -> :none
crash-improper-stack: RESCUE(FunctionClauseError)  [caught by outer try/rescue]
unknown: OK -> :none
```
**Default inspect opts bound output** (config default `translator_inspect_opts: []`):
```
list inspect bytes (default opts): 897      # from a 1,000,000-element list
binary inspect bytes (default opts): 4105   # from a 5,000,000-byte binary
```

## Risks / caveats
- **Resource consumption is config-gated.** `translator_inspect_opts: [limit: :infinity]` (or huge `printable_limit`) would let a large attacker crash term inflate memory before the formatter truncates. Operator misconfiguration footgun, not a defect in this file; defaults are bounded (verified above). A pathologically *deep* nested term is a general `inspect` concern shared by all inspect callers.
- The file was driven by compiling directly against installed Elixir 1.21.0-dev / OTP 29 stdlib; the translator depends only on standard `Exception`/`Application`/`Keyword`/`Map`/`Macro` modules.

### Ruled out
- **Uncaught crash cascade** — every clause runs under `Logger.Utils.translator/2`'s `try/rescue` (read this pass: `utils.ex:19–55`; the `rescue e ->` at 34–41 converts any exception into a `{:string, chardata}` event, and the `else` at 42–54 maps `:none`→`:ignore`, `:skip`→`:stop`). Malformed reports (non-3-tuple `reason`, absent `:dictionary`, improper-list stacktrace) raise ordinary exceptions caught there.
- **Axes examined:** exception (rescued), throw/exit escape (none observed in driven runs). **Not examined:** none.
- **`Application.format_error/1` raw-output path (lines 96/208)** — now examined by execution across reason shapes (see finding). Recorded as part of the log-injection finding, not dismissed.
- **Axes examined:** binary reason, atom reason, `{exception, stacktrace}` reason, `{:bad_return, …}` reason, `{:EXIT, {exc, stack}}` reason — pushed each through `Application.format_error/1` and compared escaped-vs-raw output. **Not examined:** none.
- **`Exception.format_banner/3` raw-output path** — now examined by execution (see finding).
- **Axes examined:** `%RuntimeError{message: <LF+ESC>}` pushed through `format_banner(:error, exc, [])`, output bytes inspected. **Not examined:** none.
- **`crash_debug_key/1` `case` without catch-all (609–621)** — only reachable at `:debug` min-level for a debug key outside proc_lib's fixed set; an unexpected key raises `CaseClauseError`, caught by the outer rescue (degraded log line, no cascade). Keys come from proc_lib's fixed report format, not attacker-shapeable.
- **Axes examined:** unknown-key input (→ CaseClauseError, rescued), min-level gate (`:debug` only). **Not examined:** none.
- **Strict destructuring matches** (gen_statem `reason: {kind, reason, stack}` at 332; `report_crash`'s `{{kind, reason, stack}, _}` at 522) — crash on unexpected shape, caught by the outer rescue; report shapes fixed by OTP.
- **Axes examined:** malformed shape (→ MatchError/FunctionClauseError, rescued). **Not examined:** none.
- **`elem(report_reason, 0) == :undef` guard (120)** — `elem/2` in a `when` guard silently fails the clause on a non-tuple (no exception), falling to `_ -> []`.
- **Axes examined:** non-tuple report_reason (guard fails safely to else-branch). **Not examined:** none.
- **`format_reason/1` stacktrace formatting (658–669)** — wraps `format_stacktrace` in `try/catch :error, _`, so malformed "stacktrace" entries (including improper lists) degrade to `format_stop` rather than crashing.
- **Axes examined:** malformed/improper stacktrace (caught, falls to `format_stop`). **Not examined:** none.
- **`user_metadata` from `$logger_metadata$` (532)** — sourced from the crashing process's own dictionary via `Keyword.get(dictionary, :"$logger_metadata$", %{})`, not a cross-tenant injection channel.
- **Axes examined:** source of dictionary (crashing process's own `crashed[:dictionary]`). **Not examined:** none.

## Did I verify this?
- **`Logger.Utils.translator/2` outer rescue** — VERIFIED by reading `utils.ex:19–55` this pass (rescue clause + else mapping). ✅
- **`Application.format_error/1` escaping behavior (lines 96/208)** — VERIFIED by execution: escaped for binary/atom/`{:bad_return,…}`/`{:EXIT,…}` reasons; **raw LF/ANSI for `{exception, stacktrace}` reasons**. This is the newly-audited axis; folded into the log-injection finding. ✅
- **`Exception.format_banner/3` raw output** — VERIFIED by execution (real LF + ESC bytes in output). ✅
- **`inspect` escapes control chars** — VERIFIED by execution (escaped `\n`/`\e`). ✅
- **Default inspect opts bound output** — VERIFIED by execution (897 bytes / 4105 bytes on 1M-element list / 5M-byte binary). ✅
- **No atom/eval/path/command/deserialization sinks** — VERIFIED by read + grep. ✅

### Dropped in re-review
None. All prior ruled-out entries and caveats are preserved. The one change is that the first-pass "log injection … not this file's defect" caveat — flagged for the hedges "outside this file" and "did not exhaustively [audit `Application.format_error/1`]" — has been **upgraded to a Low-confidence log-injection finding** after executing both `Exception.format_banner/3` and `Application.format_error/1` and confirming raw control characters reach the output from attacker-influenceable exception content, including the previously-unaudited `{exception, stacktrace}` path through `Application.format_error/1`.
