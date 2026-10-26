# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/system.ex
<!-- sha256: dfd18008aad96758ec9dea0c97c0c95379101d62968921b113a0068e32618b2e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:52:08Z_

## Summary

After a full read and targeted runtime testing of the built Elixir toolchain, `system.ex` appears correct on every documented public path I exercised; no genuine, reachable bug was found.

## Bugfixes

No findings

## Risks / caveats

- I verified behaviour against `/opt/elixir` (reported `1.21.0-dev`, built from this source tree). I did not exercise the Windows (`{:win32, _}`) branches of `do_shell/2`/`shell/2` — that OS path was reasoned about only, not run.
- Compile-time macros (`get_version`, `get_revision`, `get_date`, `get_otp_release`) were reasoned about and observed through their runtime consumers (`version/0`, `build_info/0`), which produced well-formed output; I did not re-drive them at compile time in isolation.
- Signal trapping (`trap_signal/2,3`, `untrap_signal/2`, `SignalHandler`) was read but not exercised end-to-end (it mutates process-global OS signal handlers); the review of it is `asserted`.

### Ruled out

- **`get_env/2`, `fetch_env/1`, `fetch_env!/1` doctests use `"PORT" => "4000"` and would fail if executed.** Not a finding: these `iex>` examples are never collected as doctests — a repo-wide search (`grep -rn "doctest System" . --include=*.exs`) finds no `doctest System` anywhere, only `doctest SystemLimitError`. They are illustrative placeholders depending on external env state, consistent with the module's other environment-dependent examples. Basis: `verified` (grep of the whole repo test tree).
- **`do_port_line/4` drops the trailing partial line (buffer not flushed on `:exit_status`) when output has no final newline.** Not a finding: in Erlang line mode the trailing `{:noeol, data}` arrives *after* `{:exit_status, _}`, and the nested `receive ... after 0` flushes it. Verified: `System.cmd("printf", ["hello"], into: [], lines: 1024)` → `{["hello"], 0}` (the "hello" is preserved), and `printf "hello\nworld"` → `{["hello", "world"], 0}`. Basis: `verified`.
- **Deprecated time-unit path (`:seconds`/`:milliseconds`/… → `warn/2` → `IO.warn_once/3`) could crash on an undefined function.** Not a finding: `IO.warn_once/3` exists (`io.ex:415`), each `warn(unit, replacement)` call passes the matching atom, and the conversion is correct — `System.convert_time_unit(1000, :milliseconds, :second)` emitted the deprecation warning and returned `1`. Basis: `verified`.
- **`build/0` / `build_info/0` string format vs. the documented `%{build: ..., date: ..., ...}` shape.** Not a finding: `System.build_info()` returned a map matching the documented keys and the `"<version> (compiled with Erlang/OTP <n>)"` build-string format. Basis: `verified`.
- **`halt/1` and `stop/1` guards accept negative integers while `@spec` says `non_neg_integer`.** Not a finding: reaching `:erlang.halt(-1)` requires the caller to violate the documented `@spec` (out-of-scope caller misuse), and it is not a crash on valid input. Basis: `asserted`.
- **`strip/1` regex `"^[\s\r\n\t]+|[\s\r\n\t]+$"`.** Not a finding: in an Elixir double-quoted string `\s` is a literal space, so the class is `[ \r\n\t]`; trimming works, and it feeds only compile-time constants (VERSION/git revision). Basis: `asserted` (plus the correct `revision: ""` / clean `version` observed at runtime).
- **`user_home/0` passes `native_name_encoding()` as both in- and out-encoding to `:unicode.characters_to_binary/3`.** Not a finding: this matches the intended round-trip of the platform-encoded home charlist and is the established behaviour. Basis: `asserted`.

