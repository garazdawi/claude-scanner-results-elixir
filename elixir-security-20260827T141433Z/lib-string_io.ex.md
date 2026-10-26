# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/string_io.ex
<!-- sha256: aed67b04be026d2b668d6d0d938225c57c45d961032608cf91e8b131160819c9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:41:50Z_

# Security review — `lib/elixir/lib/string_io.ex`

## Summary
`StringIO` is a small, well-behaved IO-device GenServer; the one real defect is that the `get_until` read path crashes the device on malformed UTF-8 in the *unread remainder* of the wrapped string — a content-triggered DoS that its sibling read paths (`get_chars`, `get_line`) handle gracefully.

## Security

#### `CSCAN-string_io-getuntil-unicode-crash`
- **Finding ID:** `CSCAN-string_io-getuntil-unicode-crash`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/elixir/lib/string_io.ex:424` (`get_until/7`, via `binary_to_list/2` at `:435`)
- **Class:** Resource consumption (availability DoS) / Unicode and text
- **Issue:** When a `get_until` request completes a line and the collector returns `{:done, result, extra}`, the remaining unread input is converted eagerly with `extra ++ binary_to_list(rest, encoding)`. For `:unicode`, `binary_to_list/2` is `String.to_charlist(rest)`, which **raises `UnicodeConversionError`** if `rest` contains invalid UTF-8. `rest` is the bytes *after* the newline and is never validated (`bytes_until_eol` only validates up to the end-of-line it found). The device process crashes. This is inconsistent with `get_chars` (returns `{:error, :invalid_unicode}`) and `get_line` (returns `{:error, :collect_line}`), which absorb the same malformed input without dying (`string_io.ex:358,381`).
- **Impact / threat:** The wrapped string is untrusted data (`StringIO.open/1` is documented to accept an arbitrary `string`, and the module documents the pid as a general IO device). A consumer that reads it through the `get_until` protocol — e.g. Erlang's documented `:io.fread/3` / `:io.get_until/*` — can be made to crash the device with a payload as small as `"1\n" <> <<0xFF>>`: a valid first line followed by an invalid-UTF-8 byte. The read returns `{:error, :terminated}` and the device is gone. Trust boundary crossed: attacker-controlled bytes → `StringIO.open` → documented IO-device read. (Note: Elixir's own `IO` module never emits `get_until`, so exposure is limited to Erlang `:io` consumers, which is why this is Low rather than Medium.)
- **Evidence:** `verified`.
  - `elixir -e` driving `StringIO.open("1\n" <> <<0xff>>)` then `:io.fread(pid, "", ~c"~d")` produced:
    ```
    ** (UnicodeConversionError) invalid encoding starting at <<255>>
        (elixir) lib/string.ex:3002: String.to_charlist/1
        (elixir) lib/string_io.ex:424: StringIO.get_until/7
    ...
    io.fread result: {:error, :terminated}
    DEVICE DOWN reason: {:function_clause... UnicodeConversionError ...}
    ```
  - Control: the same input via `IO.gets`/`IO.read` returned `{:error, :collect_line}` / `{:error, :invalid_unicode}` with `device still alive? true` — confirming only the `get_until` path crashes.
  - Control: identical input with `encoding: :latin1` does not crash (`:erlang.binary_to_list/1` accepts any bytes), confirming the defect is the `:unicode` `String.to_charlist` conversion of unvalidated `rest`.
- **Suggested fix:** Convert `rest` defensively instead of raising. Guard the `rest` conversion so invalid UTF-8 yields the existing `:error` return (mirroring `get_line`/`get_chars`) — e.g. use `:unicode.characters_to_list/2` and fall through to `{{:error, :collect_line}, state}` (or `:error`) when it returns a `{:error/:incomplete, _, _}` tuple, rather than calling `String.to_charlist/1`.

#### `CSCAN-string_io-open3-close-noproc`
- **Finding ID:** `CSCAN-string_io-open3-close-noproc`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/elixir/lib/string_io.ex:72-76` (`open/3`, the `after` clause)
- **Class:** Concurrency (lifetime / error masking)
- **Issue:** `open/3` runs `{:ok, {_input, _output}} = close(pid)` in an `after` block. If the device has already died while `function.(pid)` ran, `close/1` (`GenServer.call(pid, :close)`) exits with `{:noproc, ...}`. Because it is in `after`, this exit *replaces* the function's own result/exception and propagates out of `open/3`.
- **Impact / threat:** Any mid-function device death — including the content-triggered crash above — turns a well-behaved `open(untrusted, fn pid -> … end)` into a confusing `** (exit) {:noproc, {GenServer, :call, [pid, :close, 5000]}}`, masking the real cause and making the failure harder to handle/diagnose. The caller used the documented API correctly. It also blocks the `after` from returning the buffers the doc implies it collects.
- **Evidence:** `verified`. `StringIO.open("1\n" <> <<0xff>>, fn pid -> :io.fread(pid, "", ~c"~d") end)` yielded `{:caught, :exit, "{:noproc, {GenServer, :call, [#PID<...>, :close, 5000]}}"}` — the device crash surfaced only as an opaque `:noproc` exit from `open/3`.
- **Suggested fix:** Make cleanup tolerant of a dead device — e.g. match `close/1`'s result loosely and/or wrap the `after` close in a `try/catch :exit` so the function's original result or exception is preserved rather than overwritten by `:noproc`.

### Ruled out
- **`apply(mod, fun, args)` in `io_request`/`get_until` (lines 239, 247, 405, 419) — arbitrary code execution.** `verified`: sending a raw `{:io_request, …, {:put_chars, :unicode, :erlang, :atom_to_binary, [:INJECTED]}}` did execute in the device process. But this is the standard Erlang IO protocol (`{put_chars, …, M, F, A}`, `{get_until, …, M, F, A}`); Elixir's `IO.*` and Erlang's `:io.*` place untrusted *content* only as data arguments, never as `M`/`F` (confirmed: `IO.puts` emits `{put_chars, unicode, :io_lib, format, …}`, `:io.fread` emits `{get_until, …, :io_lib, :fread, …}`). Reaching it with attacker-chosen `M`/`F` requires crafting a raw message, i.e. an attacker who already has message-send/code-exec in the VM at the same privilege. Identical to every OTP IO server; not reachable via the documented public API with untrusted content.
- **Atom creation / `String.to_atom` / `Module.concat`.** None present (whole-file read).
- **File / path / command / archive / deserialization / crypto / network sinks.** None present; `{:file_request, …}` is explicitly refused with `{:error, :enotsup}` (`:202`).
- **`state_after_read` prompt handling (`:463`, `IO.chardata_to_string(prompt)` + `:binary.copy(_, count)`).** The `prompt` and `capture_prompt` opt are set by the *reading/opening* code, not by the untrusted wrapped string; malformed `prompt` violates the caller's own contract. Not an untrusted-content boundary.
- **Dead defensive code.** The invalid-UTF-8 guards (`split_at/3` `:358`, `bytes_until_eol` `:480`) and the `count >= 0` / catch-all `io_request` clauses are all reachable — `verified` that `IO.read` on invalid input returns `{:error, :invalid_unicode}` and `IO.gets` returns `{:error, :collect_line}`.

## Risks / caveats
- **Reachability scope of the `get_until` crash:** exercised here through Erlang `:io.fread/3` (a documented IO-device consumer, and the reason `get_until` exists in this module). No function in Elixir's own `IO`/`IEx` emits `get_until` (grep-verified across `lib/elixir/lib` and `lib/iex/lib`), so an application is exposed only if it (or a dependency) drives a `StringIO` through the Erlang `:io` `get_until`/`fread`/scan family over attacker-controlled content. In that setting the severity is effectively Medium.
- **Residual, not filed as findings:** (1) output accumulates via `state.output <> string` (`:311`) and, with `capture_prompt: true`, the prompt is copied `count` times per `get_until` request via `:binary.copy` (`:464`) — both are memory-growth patterns driven by the *reader/opener*, not by the wrapped string, so they are caller-controlled rather than an attacker-controlled amplification. (2) A caller-supplied `get_until` collector that never returns `{:done, …}` on `:eof` loops forever (`:404-411`), but the collector is the reader's own function. (3) The inherent IO-protocol `apply` runs at the device's privilege for anyone able to message the pid — inherent to the protocol, not fixable in this file.
- I could not exercise every Erlang `:io` entry point that emits `get_until`; I verified `:io.fread` concretely and reasoned the same `rest` path applies to other `get_until` consumers. The two filed crashes are reproduced and quoted.

