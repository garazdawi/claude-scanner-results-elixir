# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/process.ex
<!-- sha256: 6cbf75268f5103ab111774eabbcee650f473e6c0c3b8ff885de20cc176fabfed -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/process.ex
<!-- sha256: 6cbf75268f5103ab111774eabbcee650f473e6c0c3b8ff885de20cc176fabfed -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (hedge sweep: "not fuzzed")_

The flagged hedge ("not fuzzed") was retired this pass by re-executing every axis that can be pushed through the sandbox (Erlang/OTP 29, Elixir 1.21.0-dev at `/opt/elixir`): `register/2` reserved-name fuzz, `send_after/4` argument order (delivery-confirmed), `sleep/1` edge inputs and guard coverage, `info(pid, :registered_name)` three shapes vs. raw `:erlang.process_info/2`, and the `get/2` `:undefined`→default conflation. Each result is quoted below from the actual run. The `defdelegate` wrappers target `:erlang`/`:proc_lib` BIFs, which have no source file in this repo's `src/` tree, so there is no in-repo neighbour to read and no way to "fuzz the wrapper" without fuzzing the BIF itself — that is out of this file's scope and is documented as a caveat, not left as a silent assumption. The single unexecutable axis — the `sleep/1` `timeout > 0xFFFFFFFF` multi-day recursion — stays `asserted`: observing even one recursion requires the first `receive after: 0xFFFFFFFF` (~49.7 days) to elapse, and the constant cannot be lowered without editing the file (forbidden).

## Summary

`process.ex` is almost entirely thin `defdelegate` wrappers plus a handful of small logic functions (`get/2`, `sleep/1`, `register/2`, `info/2`, `whereis/1`, `put/2`, `delete/1`, `get_label/1`, `nilify/1`). Every logic function with local behaviour was re-fuzzed this pass and behaves as documented. The only defect remains the low-severity unhelpful crash from the `register/2` guard.

## Bugfixes

#### `CSCAN-process-register-reserved-name-funclause`
- **Finding ID** — `CSCAN-process-register-reserved-name-funclause`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `process.ex:718-719`, `Process.register/2` (guard `when is_atom(name) and name not in [nil, false, true, :undefined]`)
- **Issue** — When `name` is one of the four documented reserved names (`nil`, `false`, `true`, `:undefined`) — each a plain atom, therefore *valid* per `@spec register(pid | port, atom) :: true` — the guard fails and the call raises `FunctionClauseError` (`no function clause matching in Process.register/2`). That message names none of the reserved values, whereas the `:erlang.register/2` failures the function *does* catch (lines 721-732) are re-raised as `ArgumentError` with an explanatory message.
- **Class** — Contract violation (unhelpful crash on in-spec input)
- **Impact** — A caller passing a reserved atom name (e.g. `Process.register(pid, :undefined)`, possibly computed from data) gets an opaque `FunctionClauseError` pointing at `Process.register/2` internals rather than a message explaining the name is reserved. In-spec argument, so a valid-input crash with unhelpful diagnostics. Rare and low-cost, but genuinely worse than the sibling failure paths, which are documented (lines 693-705) as producing `ArgumentError`.
- **Evidence** — `verified` (re-fuzzed this pass; all four reserved names plus a control atom and a non-atom, output quoted verbatim from the run):
  ```
  register(nil)        -> FunctionClauseError: no function clause matching in Process.register/2
  register(false)      -> FunctionClauseError: no function clause matching in Process.register/2
  register(true)       -> FunctionClauseError: no function clause matching in Process.register/2
  register(:undefined) -> FunctionClauseError: no function clause matching in Process.register/2
  register(:ok_atom)   -> true          # control: valid atom registers
  register("str")      -> FunctionClauseError   # non-atom, out of spec
  ```
- **Suggested fix** — Add a `register/2` clause (or move the reserved-name check into the body) that raises `ArgumentError` with a message like `"cannot register a process under the reserved name #{inspect(name)}"`, mirroring the explanatory `ArgumentError`s already produced in the `catch` block.

## Ruled out

- **`send_after/4` argument order (`process.ex:406-408`, `:erlang.send_after(time, dest, msg, opts)`)** — `verified` (executed this pass). Scheduled `Process.send_after(self(), :ping, 10)`; the call returned a reference (`is_reference/1` true) and `:ping` was received within the timeout (`send_after delivered :ping`) — a swapped `time`/`dest` order would not deliver to `self()`. Confirms `dest`/`msg`/`time` are threaded to the BIF in the correct reordered position. `cancel_timer/2`/`demonitor/2` default-arg delegations resolve to existing `:erlang` arities.
  - Axes examined: argument order (delivery-confirmed), return type. | Not examined: none.

- **`info(pid, :registered_name)` shape normalization (`process.ex:893-899`)** — `verified` (re-executed against raw `:erlang.process_info/2` this pass, side-by-side). All three real return shapes confirmed: raw `[]` (unnamed live) → `{:registered_name, []}`; raw `{:registered_name, :named_one}` → passed through unchanged; raw `:undefined` (dead pid) → `nil`. Output quoted: `raw unnamed=[] -> {:registered_name, []}`, `raw named={:registered_name, :named_one} -> {:registered_name, :named_one}`, `dead raw=:undefined -> nil`.
  - Axes examined: unnamed-live, named-live, dead-pid. | Not examined: none.

- **`get/2` / `put/2` / `info/2` `:undefined`→default conflation (`process.ex:129-134`, `901-903`)** — `verified` (executed this pass). Stored a literal `:undefined` in the process dictionary; `Process.get(:k, :DEFAULT)` returned `:DEFAULT`, identical to the missing-key result — confirming the conflation is present and is faithful preservation of `:erlang.get/1` semantics (`:undefined -> default` at line 131), not a defect introduced here. `Process.info(dead_pid, :messages)` returned `nil` via the general `nilify` clause (line 902). Output quoted: `get(:k,:DEFAULT) stored :undefined -> :DEFAULT`, `get(:missing,:DEFAULT) -> :DEFAULT`, `info(dead,:messages) -> nil`.
  - Axes examined: stored-`:undefined` vs missing key, dead-pid general `info`. | Not examined: none.

- **`sleep/1` chunked recursion arithmetic (`process.ex:330-338`)** — `verified` for small/edge inputs and guard coverage; `asserted` for the multi-day chunking boundary. Executed: `sleep(0) -> :ok`, `sleep(1) -> :ok` (second clause), `sleep(-1) -> FunctionClauseError` and `sleep(1.5) -> FunctionClauseError` (both outside the `non_neg_integer | :infinity` spec, correctly rejected by both guard heads). For `timeout > 0xFFFFFFFF` the first clause (line 330-332) sleeps `@max_receive_after` then tail-recurses on `sleep(timeout - @max_receive_after)`; the subtraction strictly decreases and terminates in the `timeout >= 0` clause (line 334-338), in tail position — read from the two heads and the constant, but **not executed**.
  - Axes examined: zero, small positive, negative, float, guard coverage. | Not examined: the `timeout > 0xFFFFFFFF` chunked-recursion path. Observing even one recursion requires the first `receive after: 0xFFFFFFFF` (~49.7 days wall-clock) to elapse, and `@max_receive_after` cannot be lowered without editing the file (forbidden this pass). Re-implementing the arithmetic in a separate function would not be "the" implementation, so it cannot clear the axis. This axis keeps the entry `asserted`.

## Risks / caveats

- Behavioral tests ran against `/opt/elixir` (Elixir 1.21.0-dev, Erlang/OTP 29). The reviewed source at the `claude-scanner` path is byte-identical to the fuzzed build for the functions exercised (same sha256 in header), but the exact review-path tree was not independently rebuilt.
- The bulk of the file is `defdelegate` / "Inlined by the compiler" wrappers that are 1:1 delegations to `:erlang`/`:proc_lib` BIFs. Those BIFs are **not** source files in this repo (they are OTP runtime primitives), so there is no in-repo callgraph neighbour to read and no wrapper-local logic to fuzz — fuzzing them is fuzzing the underlying BIF, which is out of this file's scope. The wrappers with genuine local logic (`get/2`, `sleep/1`, `register/2`, `info/2`, `whereis/1`, `put/2`, `delete/1`, `get_label/1`) were fuzzed.
- The `register/2` finding is a diagnostics-quality issue, not a correctness or safety one — no wrong value is returned and no valid registration is lost.

## Did I verify this?

- `register/2` finding — **verified** (executed; four reserved names + control atom + non-atom, output quoted).
- `send_after/4` argument order — **verified** (executed; delivery-confirmed).
- `info(pid, :registered_name)` — **verified** (executed side-by-side vs raw `:erlang.process_info/2`).
- `get/2` / `info/2` `:undefined` conflation — **verified** (executed).
- `sleep/1` small/edge inputs + guard coverage — **verified** (executed); multi-day chunking boundary — **asserted** (`Not examined: >0xFFFFFFFF recursion path`, unexecutable — first chunk is ~49.7 days wall-clock, constant not lowerable without editing the file).

## Dropped in re-review

None. Every prior finding and ruled-out entry is preserved. The "not fuzzed" hedge is retired by re-executed evidence (quoted above) rather than by promoting an unverified assumption — re-fuzzing the local-logic functions surfaced no new defect, so no hedged assumption was upgraded to a low-confidence finding. The one remaining `asserted` basis (`sleep/1` multi-day boundary) is retained because its distinguishing axis cannot be executed in the sandbox; reading a neighbour module would not clear it (there is no in-repo neighbour for the `:erlang` BIFs, and reading tells you what a guard does, not which inputs diverge on the untried multi-day path).
