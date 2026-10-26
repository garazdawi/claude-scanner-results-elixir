# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/capture_server.ex
<!-- sha256: b7b97563640fcf65b59c09e404b5f1e378b444a8aeb2ab496ea9e35e9daff935 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:26:18Z_

# Review: `ExUnit.CaptureServer`

## Summary
The file is a faithful implementation of ExUnit's internal capture server; every path reachable through the documented public capture API (`ExUnit.CaptureIO`/`ExUnit.CaptureLog`) behaves correctly under test, and I found no bugfix-worthy defect.

## Bugfixes

No findings

## Risks / caveats

- **`revert_device_to_original_pid/2` can raise in its `after` block** (`capture_server.ex:196`) if `Process.register(pid, name)` fails because the *original* device process (e.g. `:standard_error`, `:user`) died while it was captured. This would crash the `CaptureServer` from inside `release_device/2`. I did not report it as a finding because (a) `original_pid` is provably non-`nil` for any stored device (a device is only recorded after `Process.whereis(name)` returned a live pid and `Process.register` succeeded), (b) the captured devices are long-lived system processes whose death mid-test is not reachable through normal use of the public API, and (c) this code is identical to upstream ExUnit. It is a latent fragility, not a bug reachable through documented use.
- `log/2` (the `:logger` handler callback) is invoked only by `:logger`, not by the documented public API, so it is out of scope; its `Task.async` + trapping-exit drain logic was reasoned through but is only exercised indirectly.

### Ruled out

- **`binary_part(output, offset, total - offset)` in `device_output` (line 66)** — potential `offset > total` / negative-length crash. Ruled out (verified): the StringIO backing a captured device only ever grows, and every ref's stored `offset` is `byte_size(output)` captured at registration time, so `offset ≤ total` always. Driving two concurrent captures on `:standard_error` returned `o1="AAABBB"`, `o2="BBB"` — the nonzero-offset slice is correct.
- **`input?` guard `when input? or input != ""` (line 135)** using `or` — ruled out: both operands are booleans (`input?` is stored boolean, `input != ""` is boolean), so `or` is correct. Verified: re-capturing a device with input returned `{:error, :input_on_already_captured_device}` and encoding change returned `{:error, {:changed_encoding, :unicode}}`, exactly as `capture_io.ex` expects.
- **`:logger.compare_levels(event.level, level)` with the `:all` sentinel default (lines 82, 242)** — ruled out (verified): `compare_levels(_, :all)` returns `:gt` and `compare_levels(_, :none)` returns `:lt`, so the `:all` default filters in all events rather than raising; log capture of `Logger.error("boom")` and nested captures both succeeded.
- **`handle_info({:DOWN, ...})` calling both `remove_log_capture` and `release_device` (lines 108–112)** — ruled out: `remove_log_capture` is a no-op for a device ref (`Map.pop(..., false)` → unchanged config; the `:ets.delete` of a non-existent key is harmless) and `release_device` is a no-op for a log ref (`Enum.find` returns `nil`), so a single `:DOWN` cleanly cleans up whichever kind of ref it was.
- **Reference-count thresholds `map_size(refs) == 1` (add) vs `== 0` (remove)** for both the log handler (lines 84, 223) and device revert (line 182) — ruled out: counts are checked *after* the corresponding put/delete, so first-in adds the global handler/registration and last-out reverts it exactly once; verified indirectly by nested `capture_log` producing correct isolated output.
- **`handle_call`/`handle_info` catch-all coverage** — no unhandled-message crash risk from the public API: every public wrapper (`device_capture_on/off`, `device_output`, `log_capture_on/off`) maps to a matching `handle_call` clause, and the only async message the server monitors for (`:DOWN`) has a clause.

