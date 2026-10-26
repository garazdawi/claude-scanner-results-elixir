# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/process.ex
<!-- sha256: 6cbf75268f5103ab111774eabbcee650f473e6c0c3b8ff885de20cc176fabfed -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:15:48Z_

All behavioral checks confirm my reading: the `register/2` reserved-name guard fires and rejects all four reserved names, `send_after/3` maps its reordered arguments correctly, `get/2` performs no atomization, and `sleep/0` returns cleanly. Here is the report.

---

# Security review — `lib/elixir/lib/process.ex`

## Summary
`Process` is a thin, faithful wrapper over Erlang's process/BEAM primitives; every security-relevant sink shape from the inventory is either absent or reduces to documented behavior fully controlled by the caller, so there are no defects reachable through its public API.

## Security

`No findings`

This file contains no code execution of externally-derived program text, no string→atom conversion, no file/path/command/archive/deserialization/crypto/template/regex sinks, and no cross-process shared-state writes derived from untrusted input. The functions that *do* execute code (`spawn/2`, `spawn/4`, `hibernate/3`) or send signals (`send/3`, `send_after/4`, `exit/2`) run exactly the MFA/message/reason the caller hands them — that is the documented contract, there is no "safe" variant to prefer, and no untrusted-data path is introduced by this module.

Note on scope: the standing threat model and Unicode/UAX-31 obligation checklist in my context describe the compiler front-end under `lib/elixir/src` (tokenizer/interpolation/parser). This file is `lib/elixir/lib/process.ex` — a standard-library module — so none of those Unicode identifier/bidi/NFC obligations apply to it. I confirmed by full read that no tokenizer, escape-decoding, or normalization code exists here.

### Ruled out

- **`spawn/2`, `spawn/4`, `hibernate/3` — code execution (CWE-94).** Basis: verified by full read. These delegate to `:erlang.spawn_opt`/`:erlang.hibernate` and run the caller-supplied fun or `mod/fun/args`. Running attacker-chosen code is the entire documented purpose; a caller passing untrusted MFA is misusing the primitive, and there is no confined alternative this file could use instead. Not a defect in `process.ex`.
- **Atom creation / atom-table exhaustion (CWE-770).** Basis: verified. `register/2`, `unregister/1`, `whereis/1`, `flag/2,3`, and `monitor/1` all require an atom (or pid/port) that already exists in the term; grepping the full file shows no `String.to_atom`, `binary_to_atom`, `Module.concat`, or any bytes→atom conversion. The `get/2` + `register/2` test drove these paths and none atomized a string.
- **`send/3`, `send_after/4`, `exit/2` — message/signal injection.** Basis: verified by full read + `send_after` runtime test (arguments reorder to `:erlang.send_after(time, dest, msg, opts)` correctly and the message is delivered). Destination, payload, and exit reason are caller-controlled by contract; delivering them is documented behavior, not a vulnerability introduced here.
- **`sleep/1` — resource-consumption DoS on huge integers.** Basis: reasoned from the code. For `timeout > 0xFFFFFFFF` the first clause loops via `sleep(timeout - @max_receive_after)`, but each iteration first *blocks* in `receive after: 0xFFFFFFFF` (~49.7 days), so this is not a CPU busy-loop, and the recursive call is in tail position (proper BEAM tail call → constant stack). No unbounded CPU or stack growth. `sleep(0)` returns `:ok` as expected.
- **`register/2` reserved-name guard — dead/ineffective defensive check.** Basis: verified. The guard `name not in [nil, false, true, :undefined]` is reachable and fires: all four reserved names raise `FunctionClauseError` before reaching `:erlang.register`. The `catch` arms correctly convert `:badarg` into an `ArgumentError` (with a distinct cross-node message when `node(pid_or_port) != node()`); the `node/1` call inside the catch guard cannot leak an exception because guard evaluation failures fall through to the second clause.
- **Process dictionary (`get/1,2`, `put/2`, `delete/1`, `get_keys/0,1`) — shared-state poisoning.** Basis: verified by full read. The process dictionary is process-local; there is no cross-process or VM-global write here, so no one-caller-poisons-another surface.
- **`info/2` with `:dictionary` / `:registered_name` — information disclosure.** Basis: verified by full read. This exposes only what `:erlang.process_info/2` already exposes for a local pid; the wrapper merely maps `:undefined`→`nil`. No new disclosure introduced.
- **`get_label/1`, `set_label/1` — log injection.** Basis: verified by full read. They delegate to `:proc_lib`; the label is an arbitrary term rendered (inspected) by log/Observer tooling downstream, with no string interpolation into a sink in this file.

## Risks / caveats
- The `sleep/1` behavior for `timeout > 2^32` was reasoned from the constant and tail-call position, **not** run to completion — a genuine reproduction would block ~49 days per iteration, so that one ruleout is `asserted` rather than runtime-`verified`. The reasoning (blocking `receive after` per iteration, tail-recursive call) is straightforward and I am confident in it.
- I treated the *caller* as trusted for MFA/destination/reason arguments, per the review's trust model (the library caller is not the attacker). Findings would only arise if this file itself transformed untrusted bytes into one of those sinks, which it does not.
- The `:undefined`↔`nil` conflation in `get/2`, `put/2`, `delete/1`, and `whereis/1` (a caller that stores the literal atom `:undefined` gets `nil` back) is a **correctness** quirk, not a security issue, and is explicitly out of scope for this review.
- Scope was limited to this single file as instructed; the actual behavior of the underlying `:erlang`/`:proc_lib` primitives was assumed to match their OTP documentation and not independently audited.

