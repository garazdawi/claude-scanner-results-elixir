# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/capture_io.ex
<!-- sha256: f85e54c83ae5fd0c44f9e0dd3390a8db62477c855d86863743cf1fac33dc88ec -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:23:19Z_

## Summary

`ExUnit.CaptureIO` is a clean, well-structured capture module; the only defect found is a `StringIO` process leak on the PID-capture path when setting the target's group leader fails.

## Bugfixes

#### `CSCAN-capture_io-stringio-leak-on-gl-set-failure`
- **Finding ID** — `CSCAN-capture_io-stringio-leak-on-gl-set-failure`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `capture_io.ex:271-287`, `do_with_io/3` (PID clause)
- **Class** — Resource leak
- **Impact** — Each `capture_io(pid, fun)` / `with_io(pid, fun)` call whose target `pid` is not alive at capture time leaks one `StringIO` process (plus its ETS/registration footprint) for the lifetime of the caller's VM, and reports only a bare `** (ArgumentError) argument error` that tells the caller nothing about what went wrong. In a long-running test suite that repeatedly captures a monitored/short-lived worker pid (e.g. `capture_io(GenServer.whereis(Srv), ...)` where `Srv` has just died), these processes accumulate.
- **Evidence** — `verified`.

  The capture group-leader device is opened *before* the `try`:
  ```elixir
  {:ok, capture_gl} = StringIO.open(input, ...)
  try do
    Process.group_leader(pid, capture_gl)   # raises if pid is dead
    do_capture_gl(capture_gl, fun)          # only place capture_gl is ever closed
  after
    Process.group_leader(pid, original_gl)  # restores GL only; never closes capture_gl
  end
  ```
  If `Process.group_leader(pid, capture_gl)` raises, `do_capture_gl/2` is never reached, so the freshly-opened `StringIO` is never closed; the `after` clause restores the group leader but does not close it.

  Ran:
  ```
  $ elixir /tmp/t3.exs
  alive: false
  rescued: %ArgumentError{message: "argument error"}
  proc delta: 1
  ```
  A dead pid → `ArgumentError` → process count permanently up by 1 (the leaked `StringIO`). A control run where `fun` itself raises (`/tmp/t2.exs`) showed `proc delta: 0`, confirming the normal error path closes the device correctly and isolating the leak to the group-leader-set failure.
- **Suggested fix** — Guard the device so it is always closed when it never becomes the active group leader, e.g. wrap the `Process.group_leader(pid, capture_gl)` set + capture in a `try/rescue` that closes `capture_gl` before re-raising, or move `StringIO.open` inside a `try` whose `after` closes the device unless ownership was successfully handed to `do_capture_gl`. Optionally raise a message naming the dead pid instead of the opaque `argument error`.

## Risks / caveats

- The leak is reachable only when the supplied `pid` is not alive at the moment of capture. The `@doc` describes capturing "the group leader of another process" and shows `capture_io(GenServer.whereis(MyServer), ...)`, so a pid that has died between `whereis` and the capture is a plausible—if narrow—real input rather than pure misuse; this is why it is reported at Low severity. If the reviewer's team treats "pid must be alive" as an implicit contract, this collapses to a hygiene nit.
- The atom/named-device path (`:stderr`, arbitrary registered devices) was not exercised at runtime because it requires a running `ExUnit.CaptureServer`; I reviewed it statically only (`do_with_io/3` atom clause). Its `try/after` closes the capture via `device_capture_off(ref)` on every exit including throw/exit, and its three error branches raise informative messages, so I found nothing there — but that branch is `asserted`, not `verified`.

### Ruled out
- **`do_capture_gl/2` error asymmetry** — catches `kind, reason` (covers `:error`/`:exit`/`:throw`), closes the `StringIO`, and re-raises with the original stacktrace; the success branch also closes it. Verified no leak and correct re-raise on `fun` raising (`/tmp/t2.exs`, `proc delta: 0`, group leader restored).
- **Doctests (`capture_io.ex:102-131`)** — ran all four `:stdio` doctests plus `with_io`; outputs match exactly (`"john"`, `{4, "a\nb\n"}`, `"> this is input"`, `"this is input"`). The `:stderr` doctests use `=~`/`==` against literal output and are consistent with the documented shared-device behavior.
- **Clause dispatch / spec agreement for `capture_io`/`with_io` arities 1–3** — every documented arg shape (atom, pid, binary input, keyword options) routes to a matching head, and `map_dev/1` is applied on all atom paths (including via the 2-arity → 3-arity delegation). No shape in the `@spec`s is left without a clause. Verified by exercising the atom/binary/list/2-tuple-result paths above.
- **`capture_prompt` ignored for non-`:stdio` devices** — consistent with the documented "For IO devices other than `:stdio`, the option is ignored"; the option is read only in the PID (self/`:stdio`) path.

