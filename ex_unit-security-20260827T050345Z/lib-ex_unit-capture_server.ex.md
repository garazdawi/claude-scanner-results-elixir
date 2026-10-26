# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/capture_server.ex
<!-- sha256: b7b97563640fcf65b59c09e404b5f1e378b444a8aeb2ab496ea9e35e9daff935 -->

_Analysed on 2026-08-27T05:14:11Z_

I've now confirmed the full trust model. Let me record the conclusion.

I traced every parameter that reaches `CaptureServer` back through its only documented public callers:

- `capture_io/1,2,3` → `device_capture_on(name, encoding, input)` / `device_output` / `device_capture_off`: `name`/`encoding`/`input` are arguments the test author passes to `capture_io` (`capture_io.ex:161-187`) — trusted test code.
- `capture_log/1,2` & `with_log/2` → `log_capture_on(pid, string_io, opts)` / `log_capture_off`: `opts` (`:level`, `:formatter`) are developer-supplied options (`capture_log.ex:88,112`) — trusted test code.
- `log/2` is a `:logger` handler callback invoked by the logging subsystem; its `event` can carry arbitrary logged data, but the only sinks are a developer-chosen formatter and a `send` of chardata into a `StringIO`.

No parameter that crosses into a dangerous sink originates outside trusted test code. The file contains no atom creation from bytes, no `binary_to_term`, no code-eval, no path/file/command/network operations, and no template interpolation into a foreign interpreter.

## Summary
`ExUnit.CaptureServer` is an internal (`@moduledoc false`) GenServer that serialises global IO-device and Logger capture on behalf of `ExUnit.CaptureIO`/`CaptureLog`; every value it processes enters through those two public capture APIs from trusted test code (or the logger infrastructure), and no untrusted-input path reaches a code-exec, atom-creation, deserialisation, path, command, or injection sink — I found no security defect.

## Security

No findings

### Ruled out

- **`:public` ETS table + `formatter_mod.format/2` module dispatch (`capture_server.ex:42`, `:237-255`)** — the capture table is created `:named_table, :public`, and `log/2` reads rows and dispatches `formatter_mod.format(event, formatter_config)` on a module atom stored there. In normal use `formatter_mod` is the developer's own `{module, config}` from `capture_log` opts, or `Logger.default_formatter/1` → `Logger.Formatter` — both trusted. Injecting a hostile module would require a co-located process to call `:ets.insert(ExUnit.CaptureServer, …)`, which already presupposes arbitrary code execution in the same VM; that is a co-tenant poisoning scenario, not a privilege crossing, and is explicitly out of scope in this folder's threat model. Not a finding.
- **`:formatter` module dispatch as behaviour dispatch on external module name (`capture_server.ex:79-82,255`)** — `formatter_mod` is externally *chosen* only in the sense that the test author picks it in `capture_log(formatter: {Mod, cfg})`. The author is not the attacker and already runs arbitrary code; no dependency/archive/network path supplies this value. Trusted.
- **`binary_part(output, offset, output_size)` negative-length crash (`capture_server.ex:66`)** — `offset` is `byte_size(output)` captured at `capture_existing/new_device` time, and a capturing `StringIO` buffer only grows, so `total >= offset` and `output_size >= 0` always. Even if it could underflow, the effect is a GenServer crash inside the developer's own test run (self-DoS), not a cross-boundary security issue. Not reachable and not security.
- **`Map.fetch!` on `name`/`ref` in `device_output` (`capture_server.ex:61,64`)** — a bad `name`/`ref` raises `KeyError`, but both come as a matched pair from the same trusted `capture_io` lifecycle; a mismatch is a caller-contract violation (explicitly out of scope) and at worst crashes the caller's own suite.
- **Signal/handler balance (`:logger.add_handler`/`remove_handler`, monitor demonitor)** — add/remove of the `:default` and `@name` logger handlers and the process monitors are balanced across `log_capture_on`/`log_capture_off`/`handle_info(:DOWN…)` and gated on `map_size` reaching 0/1; no leak reachable from the public API that constitutes a security defect. Robustness only.
- **`Task.async`/`await_many` fan-out in `log/2` (`:253-272`)** — task count is bounded by the number of distinct `{formatter_mod, formatter_config}` groups among active captures (i.e. by concurrent tests), not by attacker input; no resource-exhaustion lever crosses a trust boundary.

## Risks / caveats
- I treated the test author, their `capture_io`/`capture_log` options, and the code-under-test as trusted, consistent with ExUnit's threat model (the operator *is* the test author, who already has arbitrary code execution). The only genuinely external data reaching this file is log-event *content* in `log/2`, which is never interpreted here — it is handed to the configured formatter and the formatted chardata is forwarded to a `StringIO`; a crash inside a hostile/dependency-chosen formatter is a robustness concern in that formatter, not a defect in this file.
- I did not need to run reproductions because no hostile input path reaches a sink; the conclusion rests on tracing the two public callers (`capture_io.ex`, `capture_log.ex`) rather than on behavioural observation. If a follow-up wants to harden against co-tenant ETS poisoning of the formatter-dispatch row, it would make the capture table non-`:public`, but that is defence-in-depth outside the stated model, not a fix for a reachable vulnerability.

