# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/watcher.ex
<!-- sha256: 304a165dcc96b4ae5ed5a95f47b167677987523396dae088f08babafc2364d79 -->

_Analysed on 2026-08-27T08:35:44Z_

# Security review — `logger/backends/watcher.ex`

## Summary
One genuine but low-severity, deprecated-path issue: a crashing legacy `:gen_event` backend's exit reason — which can carry attacker-controlled logged data — is written raw to stderr, enabling log-line and terminal-escape injection; everything else in this small watcher operates on trusted operator config.

## Security

#### `CSCAN-watcher-exit-reason-injection`
- **Finding ID:** `CSCAN-watcher-exit-reason-injection`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `watcher.ex:60` (`handle_info({:gen_event_EXIT, handler, reason}, handler)`), via `format_exit/1` at `watcher.ex:55,86-87`
- **Class:** Template / interpolation (log-line / terminal-escape injection, CWE-117 / CWE-150)
- **Impact:** In a Logger configuration that uses the legacy `:backends` mechanism, when a supervised backend crashes while processing a log event, `Exception.format_exit/1` renders the crash reason and it is written unescaped to the node's stderr. If the crashing backend's exception message embeds attacker-controlled data that was logged (usernames, headers, request bodies forwarded via `Logger.error/1` et al.), an attacker can smuggle CR/LF to forge additional log lines (spoofing e.g. a `[error] FORGED ADMIN ACTION` entry into a captured log stream/aggregator) and raw ANSI/CSI escape sequences to manipulate an operator's terminal. It does not yield code execution or data loss — only log/terminal spoofing on a fallback error path.

- **Evidence:** `verified`. I built a real supervised `:gen_event` handler (a stand-in Logger backend), crashed it on an event carrying attacker data, and reproduced exactly what this file does with the resulting message.

Command (`/tmp/gen_ev_test.exs`, run with the elixir under review, Elixir 1.21.0-dev / OTP 29), key excerpt:
```elixir
:ok = :gen_event.add_sup_handler(:test_mgr, CrashBackend, [])
attacker = "user=admin\n2026-08-27 12:00:00 [error] FORGED ADMIN ACTION \e[31mX\e[0m"
:gen_event.notify(:test_mgr, {:log, attacker})   # backend does: raise "invalid log data: " <> attacker
receive do {:gen_event_EXIT, handler, reason} -> ... end
```
Observed — the message OTP delivers is exactly the `{:EXIT, _}` shape clause 1 of `format_exit/1` expects, and the watcher's stderr output preserves the injection:
```
=== reason matches {:EXIT, _}? ===   true
=== WHAT WATCHER WRITES TO STDERR (raw) ===
:gen_event handler CrashBackend installed in Logger terminating
** (exit) an exception was raised:
    ** (RuntimeError) invalid log data: user=admin
2026-08-27 12:00:00 [error] FORGED ADMIN ACTION [31mX[0m       <-- injected line + raw ESC bytes
        gen_ev_test.exs:6: CrashBackend.handle_event/2
        ...
```
The injected `FORGED ADMIN ACTION` appears on its own line and the `\e[31m…\e[0m` escapes are emitted as raw control bytes. (A companion run confirmed the raw rendering comes from `Exception.message/1` on the exception-with-stacktrace path; with an empty stacktrace `format_exit` falls back to `inspect` and escapes the bytes — the vulnerable path is the realistic one, since a real crash always carries a stacktrace.)

- **Reachability notes (why Low, not higher):** (1) This module is instantiated only under the **deprecated `:backends`** config; the default OTP-handler path never uses it. (2) The stderr write at line 60 is gated behind `logger_has_backends?()` returning false — i.e. it only fires when the crashing handler was effectively the last real backend, a fallback "logging is broken, print directly" branch that bypasses the normal formatter's truncation. (3) It requires a backend to crash with attacker data in its exception message; the stock console backend is fairly crash-resistant, but custom legacy backends that do `raise "...#{value}"` on bad input are the natural trigger. The data flow originates in Logger's documented public logging API, not from any direct call into this internal module.

- **Suggested fix:** Sanitize the reason before it reaches stderr — strip/escape control characters on the interpolated portion, e.g. run `format_exit(reason)` output through the same pruning Logger applies elsewhere, or `inspect` the rendered string so CR/LF/ESC are escaped (`IO.puts(:stderr, [..., inspect(IO.iodata_to_binary(["** (exit) ", format_exit(reason)]))])`), rather than emitting `Exception.format_exit/1`'s raw message bytes.

## Risks / caveats
- **Practical trigger is backend-quality-dependent.** I verified the sink and the exact OTP message shape end-to-end, but whether a *given* deployment's legacy backend actually crashes with attacker-controlled bytes in its exception message depends on that backend's code; hence medium confidence and Low severity.
- **Crash-loop DoS not counted as a separate finding.** A backend that deterministically crashes on a stream of hostile messages will cycle crash→watcher-stop→supervised-restart→`add_sup_handler`, and could hit the backends supervisor's `max_restarts` (30/3s per the folder threat model), escalating up `Logger.Backends.Internal` (`:rest_for_one`) and disabling legacy logging. The crash originates in the backend, not this file; the watcher's role is standard supervised restart, so I did not raise it against `watcher.ex`, but it compounds the injection issue on the same deprecated path.
- I could not exercise this inside a fully-booted Logger app with real `:backends` config (would require standing up the deprecated backend stack); the reproduction instead drives the identical `:gen_event` primitives and the file's exact message-construction/`format_exit` logic.

### Ruled out
- **`inspect(handler)` (line 52) / handler & args in `init`/`terminate` (lines 15-40, 82):** `handler`/`args` come from operator `:backends` config, not attacker data — trusted. `inspect` also escapes control chars regardless.
- **`init/1` self-recursion (lines 38-39):** bounded delete-then-add pattern (second `delete_handler` returns `{:error, :module_not_found}` and proceeds); driven by config, not external input — no unbounded loop.
- **`logger_has_backends?/0` → `:gen_event.which_handlers/1` (line 72):** no external data; the check-then-act only decides whether to print a diagnostic and is wrapped in `try/catch` — benign, consistent with prior review of `which_handlers` races.
- **`terminate/2` `delete_handler` (line 82):** operates on the config-derived handler in state; no external path, no traversal/injection sink.

