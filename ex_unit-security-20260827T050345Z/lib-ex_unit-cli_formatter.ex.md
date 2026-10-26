# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/cli_formatter.ex
<!-- sha256: 7bf48a0d9e4df4aef730e95b1b86494a20d05bc34b429318e0a0abd722a9e21a -->

_Analysed on 2026-08-27T05:17:38Z_

Confirmed. `plural_rule/1` is a map lookup keyed by a binary and returns a binary — no atom creation from the interpolated `test_type`. And `Path.relative_to_cwd/1` is pure path-string computation used only for display, not a filesystem-operation sink. I have now traced every sink in the file.

## Summary

`ExUnit.CLIFormatter` is an internal (`@moduledoc false`) `GenServer` formatter that receives ExUnit's test-lifecycle events and renders human-readable, ANSI-colored progress/summary/failure output to the terminal; every value it handles is trusted test metadata (descriptions, module names, file paths, tags, captured logs) and the only sinks are `IO.puts`/`IO.write`, so its security posture is clean — no code-exec, atom-creation, deserialization, command, path-traversal, or network sinks exist in this file.

## Security

No findings

### Ruled out

- **Atom-table exhaustion via `pluralize_type/2`** — `pluralize_type` calls `ExUnit.plural_rule(to_string(type))`. `to_string/1` converts an existing `test_type` atom to a string (no atom creation), and `plural_rule/1` (verified at `ex_unit.ex:446`) is a `Map.get` on app-env keyed by that binary, returning a binary. No `String.to_atom`/`to_existing_atom` anywhere in the file. Not a sink.
- **Path traversal / file operations via `Path.relative_to_cwd/1`** (lines 160, 172, 250, 274, 349) — these compute a display-only relative path string; the formatter never reads, writes, deletes, or links any file. No `File.*`, `:erl_tar`, `:zip`, wildcard, or `Path.join`/`expand` on external components. Pure string manipulation for output.
- **Terminal-control-character injection via `print_logs/1` and trace/description output** (lines 254, 587–591) — captured logs and test descriptions are written to the terminal with only `\n` normalized, so ESC/control bytes pass through. This is by design: the formatter's own job is to emit ANSI escape sequences (colors), and ExUnit deliberately does not sanitize test/log content (which legitimately contains escapes). The data source is the developer's own test code / code-under-test (trusted per the folder threat model); no documented public API forwards attacker-controlled bytes here as a privilege crossing. Consistent with prior OTP terminal-escape findings only when the source is genuinely untrusted network/pre-auth data — not the case here.
- **Division-by-zero in `format_slowest_tests/2` and `format_slowest_modules/2`** (lines 306, 335) — `Float.round(slowest_us / run_us * 100, 1)` raises `ArithmeticError` if `times_us.run == 0`, crashing the formatter process. Reachability requires the developer's own `--slowest`/`--slowest-modules` CLI flag *and* a total run time measured as exactly 0 µs, which does not occur for a real run. Self-inflicted robustness edge, not a security defect; no external trust boundary crossed.
- **`inspect/1` on `name`/`parameters`/`module`** (lines 160, 163, 274, 344, 349) — `inspect` is total over built-in terms; a crash would require a developer-authored struct with a crashing custom `Inspect` impl (trusted code), and would only fault the formatter. No external input.
- **Dead/unreachable defensive code** — the `when is_binary(reason)` guards (lines 95, 102), `print_logs("")` (585), and `get_terminal_width` fallbacks (556–561) are all reachable and fall through cleanly to the catch-all `handle_cast(_, config)` (235); none are dead defenses.

## Risks / caveats

- This module is `@moduledoc false`; its effective "public API" is the ExUnit event protocol consumed as a configured formatter. I treated the event stream (`ExUnit.Test`/`ExUnit.TestModule` structs) as the entry point per the documented custom-formatter extension point. Under a strict "documented public API only" reading there is even less surface.
- Failure rendering itself (`format_test_failure/5`, `format_test_all_failure/5`) lives in `ExUnit.Formatter`; this file only prints their already-formatted output. Any inspect-time/format crash on pathological asserted terms would belong to `formatter.ex`, not here — worth confirming there if a broader formatter review is in scope.
- I did not execute a live reproduction because there is no positive vulnerability to trigger; the ruled-out items are non-security (robustness / by-design) and were settled by code reading plus the `plural_rule/1` confirmation. The div-by-zero characterization is `asserted`, but its exclusion rests on trust-boundary/reachability, not on the exact exception type.

