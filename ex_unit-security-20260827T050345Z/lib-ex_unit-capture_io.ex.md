# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/capture_io.ex
<!-- sha256: f85e54c83ae5fd0c44f9e0dd3390a8db62477c855d86863743cf1fac33dc88ec -->

_Analysed on 2026-08-27T05:15:05Z_

All edge behaviors confirmed:
- **Raise messages** use `inspect(device)` → safe representation, no injection/eval; the device is already an atom (no atom-table growth).
- **Charlist-as-input** is routed to the options branch and ignored — benign shape confusion, no sink reached.
- **Live-pid capture** restores the group leader (no leak, no lingering redirection).
- **`input` option** is inert simulated stdin, read back by the function — never a sink.

My analysis is complete. This file is a thin capture wrapper; every input originates from the trusted test author, and no untrusted (dependency/archive/network/file) data reaches any dangerous sink through the documented public API. The one genuine bug (dead-pid `StringIO` leak) crosses no trust boundary, so it falls outside the security-only scope — I report it transparently under Ruled out.

## Summary

`ExUnit.CaptureIO` is a test helper that captures a process's IO by swapping its group leader for an in-memory `StringIO` (self/pid path) or by temporarily hijacking a named IO device via `ExUnit.CaptureServer` (atom path), then restoring it. Its posture is sound for a security review: all inputs (the `fun`, `pid`, device atom, `input` string, and options) originate from the trusted test author, and no untrusted data reaches a code-exec, atom-creation, deserialization, path, command, or network sink — I found no security findings.

## Security

No findings

### Ruled out

- **`fun.()` "code execution" (lines 283, 297, 325)** — `fun` is the caller's own 0-arity function (guarded `is_function(fun, 0)`); running it is the documented purpose. No external program text is compiled/evaluated. Not a sink for untrusted input.
- **Atom creation on the device path (`map_dev/1`, `do_with_io/3` atom clause, line 269/293)** — `device` is guarded `is_atom(device)`; it is *matched/passed through*, never created from a string. No `String.to_atom`/`Module.concat`. *verified:* `capture_io(:definitely_not_registered_device_xyz, …)` raised `could not find IO device registered at :definitely_not_registered_device_xyz` — the unknown atom is looked up, not minted, and yields a clean error with no atom-table growth.
- **Interpolation in `raise` messages (lines 303, 306–314, 317–319)** — all interpolate through `inspect/1`, which emits a safe code representation; the values are caller atoms/encodings. No log/query/format-string injection sink.
- **`Process.group_leader(pid, capture_gl)` shared-state write (lines 282, 285)** — the target `pid` is a caller argument; redirecting another process's group leader is the documented, opt-in feature ("pass a pid … not safe to run concurrently"). No external-input-derived key/value; *verified:* for a live pid the `after` clause restores the original group leader (`live pid GL restored? true`).
- **Concurrency of pid capture** — non-`self()` pid capture racing on the same group leader is explicitly documented as unsafe for `async: true`; the atom path is serialized through `CaptureServer`. Documented limitation, not a defect.
- **`input`/`encoding`/`capture_prompt` options → `StringIO.open/2`** — `input` becomes inert simulated stdin read back by `fun`; *verified:* `capture_io([input: "line1\nline2\n"], …)` returned the bytes verbatim with no side effect. Cost is bounded by caller-supplied data; no amplification.
- **Charlist/binary shape confusion (`capture_io/2`, `with_io/2` list vs binary clauses)** — a charlist passed where `String.t()` input is expected is routed to the options branch and ignored; *verified:* `capture_io(~c"charlist-as-input", …)` returned cleanly. This requires the caller to violate the `String.t()` type spec and reaches no sink.
- **Dead-pid `StringIO` process leak (`do_with_io/3`, lines 276–286)** — *this is a genuine bug but out of security scope.* When a dead pid is passed, `Process.info(pid, :group_leader)` returns `nil` (fallback taken), then `Process.group_leader(pid, capture_gl)` raises `ArgumentError` *before* `do_capture_gl/2` (the only code that closes `capture_gl`) runs, and the `after` clause also raises without closing it, leaking one `StringIO` process per call. *verified:* `capture_io(dead_pid, fn -> IO.write("nope") end)` → `{:raised, ArgumentError}` with StringIO process count `before=0 → after=1` (`LEAKED? true`). Excluded because the pid is a caller-supplied argument: no dependency, archive, git ref, network, or file data can become it, so no trust boundary is crossed and accumulation is self-inflicted. Worth fixing as an ordinary robustness bug (see Risks).

## Risks / caveats

- **Scope boundary — collaborators not reviewed here.** The atom-device path delegates to `ExUnit.CaptureServer` (separate file), whose `capture_new_device/5` does `Process.unregister(name)` + `Process.register(pid, name)` to hijack an arbitrary named registration. That is caller-chosen and documented behavior, but any security question about hijacking a live named process (e.g. a registered `GenServer`/`Logger`) belongs to `capture_server.ex`, not this file. Likewise `StringIO` internals were treated as a trusted in-memory collaborator.
- **Non-security bug worth flagging to maintainers.** The verified dead-pid `StringIO` leak (and the symmetric case where a live target pid dies mid-`fun` so the `after` `Process.group_leader/2` raises and masks a successful result/re-raise) is a real robustness/cleanup defect. The minimal fix is to guard the group-leader swap/restore against a non-alive pid (e.g. skip/rescue the restore and ensure `capture_gl` is closed on the raising path). I left it out of the Security section only because it crosses no trust boundary; a general-correctness review should pick it up.
- **Assumption.** I treated the test author as the sole source of `pid`, `device`, `input`, `options`, and `fun`, consistent with the folder threat model ("nobody hostile"). If a downstream consumer were to feed genuinely attacker-controlled bytes as a *device atom* (converting a string to an atom themselves before calling), that atom-minting would live in the caller, not here — this file never performs the conversion.

