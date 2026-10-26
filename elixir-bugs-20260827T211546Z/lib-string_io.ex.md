# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/string_io.ex
<!-- sha256: aed67b04be026d2b668d6d0d938225c57c45d961032608cf91e8b131160819c9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:51:40Z_

# StringIO bug-hunting review

## Summary
The file is byte-for-byte identical to shipping Elixir 1.21.0-dev, all 9 doctests pass, and fuzzing the documented IO-device surface found no crash on valid input; the only genuine deviation is that `capture_prompt` records a single `get_until` prompt once per internal `:more` iteration rather than once per call.

## Bugfixes

#### `CSCAN-string_io-get-until-prompt-repeated-per-iteration`
- **Finding ID** — `CSCAN-string_io-get-until-prompt-repeated-per-iteration`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `string_io.ex:387-397` (`get_until/6`) feeding `count` into `state_after_read/4` at `string_io.ex:463-466`
- **Issue** — In `get_until`, `count` is incremented once for every internal `apply(mod, fun, ...)` call (`string_io.ex:407,410,421,424,427`). That accumulated `count` is passed to `state_after_read`, whose `capture_prompt: true` clause copies the prompt `count` times: `:binary.copy(IO.chardata_to_string(prompt), count)` (`string_io.ex:464`). A single `get_until` request has exactly one prompt argument, yet a request function that returns `{:more, _}` across N lines causes the prompt to be captured N times. The single-iteration paths `get_line`/`get_chars` correctly hard-code `count = 1` (`string_io.ex:369,373,328`), so only `get_until` diverges.
- **Class** — State / accumulator threading (prompt-repeat count conflated with iteration count)
- **Impact** — Callers using `StringIO` as a captured IO device (`capture_prompt: true`) and driving it with a multi-line `get_until` (reachable through the documented `:io`/erlang IO-protocol contract that `StringIO` advertises support for) get an output buffer containing the prompt duplicated once per consumed line instead of once per call. The doc states prompts "specified as arguments to `IO.get*` functions" are captured — singular per call — so the captured buffer overstates what a real terminal would have shown. Purely a fidelity/cosmetic defect in the captured output; no crash or data loss.
- **Evidence** — `verified`. With `StringIO.open("ab\ncd\nef", capture_prompt: true)` and a `get_until` function that finishes only after two newlines, `:io.request(pid, {:get_until, :unicode, ">", H, :until, [nil]})` then `StringIO.contents(pid)` yields `{"ef", ">>"}` — the single `">"` prompt captured twice (command/output above). A one-newline function yields a single `">"`.
- **Suggested fix** — Capture the prompt once per `get_until` request regardless of internal iterations, e.g. pass `1` (not `count`) as the prompt-repeat argument to `state_after_read` on the `get_until` path, while keeping `count` for any other purpose. Minimal change: in `get_until/6` call `state_after_read(state, input, prompt, 1)`.

## Risks / caveats
- **This file is identical to the installed reference** (`diff` against `/opt/elixir/lib/elixir/lib/string_io.ex` reports IDENTICAL), so any defect here is a latent upstream one, not an injected regression; the one finding above is longstanding and may be regarded as accepted behavior by the maintainers — hence medium confidence that it rises to "bug."
- I exercised the public API (`open/2`, `open/3`, `contents/1`, `flush/1`, `close/1`) and the documented IO-device protocol (`IO.read/gets/getn/write/puts/stream/binread/binstream`, `:io.get_chars/getopts/setopts/request`, `get_until`) against ASCII, multibyte-UTF-8, `\r\n`, lone `\r`, invalid-UTF-8, empty, EOF-past, and count-0 inputs; none crashed StringIO or returned a wrong shape.
- Some behaviors that look suspicious are correct-by-contract and were not reported: `put_chars` correctly rescues the `ArgumentError` that `:unicode.characters_to_binary/3` raises on bad chardata; `get_chars`/`get_line` on invalid UTF-8 return `{:error, :invalid_unicode}` / `{:error, :collect_line}` without consuming input; unbalanced `:setopts` maps fall through to `{:error, :enotsup}`.

### Ruled out
- **`IO.getn(pid, "", 0)` raises `FunctionClauseError`** — the crash is in `IO.getn/3` (`io.ex:570`) guarding `count > 0` before any message is sent; StringIO's own `{:get_chars, ...}` guard is `count >= 0` and handles 0 (returns `""`, input unchanged). Not a defect in this file. Basis: `verified` (stacktrace points at `io.ex:570`).
- **`put_chars` with an `apply` returning a non-chardata term** — `:unicode.characters_to_binary/3` raises `ArgumentError`, which the `rescue` at `string_io.ex:316-317` catches, replying `{:error, req}`; process stays alive. Basis: `verified`.
- **`get_until` `{:done, result, extra}` where `extra` is a binary** would make `extra ++ binary_to_list(rest, ...)` (`string_io.ex:424`) crash. The IO protocol requires the pushed-back leftover to be a list (same char form supplied to the function), so this needs a request function that violates its own contract — internal-interface misuse, out of scope. Basis: `asserted`.
- **`init` accepts an unvalidated `:encoding`** (e.g. `encoding: :foo`) — reachable only by passing an option outside the documented `open_opts` (`:unicode | :latin1`), i.e. a `@spec`/contract violation; and even then it degrades to `{:error, ...}` rather than crashing. Out of scope. Basis: `verified` (no crash) / contract.
- **`handle_call` handles only `:contents`/`:flush`/`:close`; `handle_info` has a catch-all** — the only calls issued are from this module's public functions, and `handle_info/2` ends with a `_message` catch-all (`string_io.ex:211-213`) plus `:DOWN` handling, so no mailbox leak or unhandled-message crash on any documented path. Basis: `verified` (message-shape inventory of the module + fuzz sweep produced no crash).

