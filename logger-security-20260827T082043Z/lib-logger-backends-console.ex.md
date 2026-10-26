# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/backends/console.ex
<!-- sha256: 217443a1bd0e306bdcebff542d8725de83755b82601199f375c9a3aabab4ff10 -->

_Analysed on 2026-08-27T08:33:51Z_

I have thoroughly traced the file and reproduced the relevant behaviors. Here is my report.

---

## Summary

`Logger.Backends.Console` is a thin, deprecated `:gen_event` backend that forwards already-formatted log chardata to an IO device; the only in-file sink that touches an attacker-influenceable value without validation is the `:ansi_color` metadata reaching `IO.ANSI.format_fragment/2`, and it is only attacker-reachable under an atypical application pattern.

## Security

#### `CSCAN-console-ansi-color-metadata-unvalidated`
- **Finding ID:** `CSCAN-console-ansi-color-metadata-unvalidated`
- **Severity:** Low
- **Confidence:** low
- **Location:** `console.ex:252-253`, `color_event/4`
- **Class:** Template / interpolation (terminal-escape injection) + Resource/availability (backend crash), via unvalidated metadata value.
- **Impact:** If an application copies untrusted data into log metadata under the key `:ansi_color` (e.g. a generic wrapper that bulk-merges request params/headers into `Logger.metadata/1`), an attacker controls the value handed to `IO.ANSI.format_fragment(color, true)`. Two effects, both reproduced: (a) a **binary** value is emitted to the terminal verbatim — raw ANSI/OSC escapes (e.g. `\e]0;…\a` window-title set, cursor moves, color spoofing) reach the operator's terminal ahead of the message; (b) an **atom** value that is not a known ANSI name makes `IO.ANSI.format_fragment` raise `ArgumentError` *inside* `handle_event/2`, which terminates the `:gen_event` handler and removes the console backend (log loss). Repeated triggers cause the `Watcher`→`Backends.Supervisor` restart loop (30 restarts / 3s) to give up, taking the legacy backend subtree down until the app restarts. The atom (DoS) variant needs the value to be an atom, which raw network bytes are not, so the practical remote effect is the terminal-escape injection.
- **Evidence:** `verified`.
  - `IO.ANSI.format_fragment(:not_a_color, true)` → `%ArgumentError{message: "invalid ANSI sequence specification: :not_a_color"}`; `IO.ANSI.format_fragment("\e]0;PWNED\a", true)` → returns `"\e]0;PWNED\a"` unchanged.
  - End-to-end through the real backend: `Logger.bare_log(:info, "msgA", ansi_color: "\e]0;VIA-ANSICOLOR\a")` printed `^[]0;VIA-ANSICOLOR^GmsgA` to the console (`cat -v`). `Logger.bare_log(:info, "msgB", ansi_color: :definitely_not_a_color)` produced `:gen_event handler Logger.Backends.Console installed in Logger terminating ** (ArgumentError) invalid ANSI sequence specification: :definitely_not_a_color` at `console.ex:253`, and afterwards `:gen_event.which_handlers(Logger)` returned only `[Logger.Backends.Config]` — the console handler was gone.
- **Suggested fix:** Do not feed an unvalidated metadata value straight to `IO.ANSI.format_fragment`. Restrict `md[:ansi_color]` to the known color atoms (e.g. `if color in @allowed_ansi, do: color, else: Map.fetch!(colors, level)`), or wrap the fragment call so an unexpected value falls back to the level color rather than raising and killing the handler. This also caps the escape-injection surface to legitimate SGR color codes.

## Risks / caveats

- **Message-content terminal/CRLF injection is real but belongs to `Logger.Formatter`, not this file.** `Logger.bare_log(:info, "attacker\e]0;PWNED\aline1\r\nFAKE injected line")` emitted the raw OSC escape and a forged second line to the terminal — *and the still-active default `:logger` handler produced the identical injection*, confirming the un-escaped `$message` handling lives in `Logger.Formatter.format`/`prune` (out of scope here). `console.ex` only wraps that output in SGR color codes and forwards it; it introduces no additional un-escaped path, so I did not raise it against this file.
- I could not manufacture a realistic remote path that sets an *atom*-valued `:ansi_color` (needed for the crash/DoS variant) from pure network bytes, which is why the finding is Low/low-confidence; the binary injection variant is the more plausible one and largely overlaps the formatter issue above.
- `configure/1`'s `Application.put_env(:logger, __MODULE__, config)` and `Logger.Formatter.compile(format)` operate on operator/developer config (via `Logger.configure_backend/2`), not attacker data.

### Ruled out
- **`Map.fetch!(colors, level)` KeyError on an unexpected level (`:warn`/`:all`/`:none`)** — not reachable. `handle_event/2` overwrites `level` with `md[:erl_level]` (`console.ex:68`), and `Logger.Backends.Handler.log/2` always sets `erl_level` to the original OTP level, which is always one of the 8 color-map keys. Verified: logging all eight levels including `:warning` through the live backend produced correct colors and zero KeyError; the collapsed `:warn` tuple element never reaches `color_event`. (A raw `:gen_event.notify` without `erl_level` could reach it, but that is an undocumented internal message, out of scope.)
- **`send(device, {:io_request, …, {:put_chars, :unicode, output}})`** — the sink that prints to the device; `output` is formatter-produced chardata whose escaping/truncation is the formatter's responsibility. No new transformation here.
- **Unbounded buffering / memory** — buffer is count-bounded by `max_buffer` (default 32) with `await_io/1` backpressure at the cap; each event is size-bounded by formatter truncation. Operator can misconfigure `max_buffer`, but that is not attacker-controlled.
- **`retry_log/2` recovery path** — reached only on IO-device `{:error, …}` replies; `:unicode.characters_to_binary/1` failures are rescued and the malformed data is emitted via `inspect(dirty)`, which neutralizes control bytes (verified: `inspect("\e]0;PWNED\a\r\n")` → `"\e]0;PWNED\a\r\n"` escaped). Pruning converges; `Logger.Formatter.prune/1` is out of file.
- **`raise` on `{:DOWN, …}` / unrecoverable IO error** (`console.ex:100,224,281`) — availability only, and the IO device (`:user`/`:standard_error`) is operator-selected, not attacker-controlled.

