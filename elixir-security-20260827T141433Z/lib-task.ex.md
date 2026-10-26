# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/task.ex
<!-- sha256: e8cf64cde2854e3463ef6480931b09e86dac7ccb3b49e0b716e8b35ba09037a4 -->
<!-- tree: 3389f8405 2026-08-25 -->

# Security review — `lib/elixir/lib/task.ex`
<!-- sha256: e8cf64cde2854e3463ef6480931b09e86dac7ccb3b49e0b716e8b35ba09037a4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28. Hedges from the first pass resolved by (a) reading the callgraph neighbour `lib/elixir/lib/task/supervised.ex` and (b) re-running the two behavioral checks in the working Elixir/OTP toolchain (`/opt/elixir/bin/elixir`)._

## Summary
`Task` is a concurrency primitive whose only "inputs" are caller-supplied functions/MFAs and BEAM messages matched by opaque monitor references/aliases. It contains no code-eval, atom-creation, file/path, command, deserialization, crypto, or template sinks, and the one security-relevant guard (the owner check) is correct. The first-pass hedges about the sibling module `Task.Supervised` (stream option validation, concurrency bounding, MFA dispatch) have now been checked directly against that file. **No security findings.**

## Security

`No findings`

### Ruled out

- **MFA execution via `start/3`, `start_link/3`, `async/3`, `async_stream/5` (Code execution).** These run caller-supplied `{module, function, args}`. The dispatch site is now confirmed: `Task.Supervised.invoke_mfa/2` calls `apply(module, fun, args)` directly (`task/supervised.ex:103-104`) with **no in-file transformation of external bytes into the module/function/args** — the tuple is threaded verbatim from the `Task.*` caller through `noreply/4`/`reply/3` (`supervised.ex:36,40,69-74`). This is the documented purpose of the module; forwarding attacker-controlled module/function names is caller misuse, analogous to the excluded "caller runs `Code.eval_string` on attacker input." **Confidence: high.** Basis: read of both files; the only doc-comment hit for "apply" in `task.ex` is at line 494; the real sink is `supervised.ex:104`.
  - *Resolves first-pass hedge* "the heavy MFA machinery in `Task.Supervised` … was not reviewed" — now read.

- **Atom creation / atom-table exhaustion.** No `String.to_atom`, `List.to_atom`, `binary_to_atom`, `Module.concat`, or `Module.safe_concat` in either `task.ex` or `task/supervised.ex`. The `is_atom/1` guards on `module`/`function_name` require already-existing atoms. **Confidence: high.** Basis: grep (no matches) + full read of both files.

- **Owner authorization check (`owner != self()`) — validation sink.** `await/2` (876), `await_many/2` (986), `yield/2` (1121), `yield_many/2` (1244), `ignore/1` (914), and `shutdown/2` (1344) all reject a non-owner caller; `shutdown(%Task{pid: nil})` (1340) routes through `ignore/1`, which also checks. Comparison is against `self()` (unforgeable). **Confidence: high.** Basis: **verified by execution** — a spawned non-owner calling `Task.await(t)` raised `ArgumentError` (`non-owner: {:rescued, ArgumentError}`), while the owner got `42`.

- **Secret leakage via `inspect` in `invalid_owner_error/1` (line 1500).** The message inspects the `%Task{}` struct, which stores only `{module, function, arity}` in `:mfa` plus `owner`/`pid`/`ref` — never the closure or argument list. **Confidence: high.** Basis: **verified by execution** — `Task.async(fn -> "SUPER_SECRET_TOKEN" end)` inspected as `%Task{mfa: {:erlang, :apply, 2}, owner: #PID<…>, pid: #PID<…>, ref: #Reference<…>}` (no secret present); `Task.async(Kernel, :+, [40, 2])` stored `mfa: {Kernel, :+, 2}` (arity, not `[40, 2]`).

- **Reply-message spoofing.** `await`/`await_many`/`yield`/`yield_many`/`shutdown` match replies only by the monitor reference/alias (`{^ref, reply}`). Within this file, a `ref` is created **only** by `build_alias/1` (1463, an `alias: :demonitor` monitor) via `async/1,3` (521) and `__alias__/1` (1453), or by `make_ref/0` in `completed/1` (559); the containing `%Task{}` struct is returned only to the calling `owner`. No code path in `task.ex` (or in `supervised.ex`, which only ever `send`s to the `reply_to`/`self()` it was handed) constructs a reply from externally-derived data. **Confidence: high** that no in-code path leaks the ref; the one residual assumption is the platform guarantee that alias references are unforgeable by an unrelated process — a BEAM property this file cannot and need not enforce. Basis: read of both files; not executable because it is a structural (not input-driven) property.

- **Resource consumption (`await_many`, `yield_many`, `async_stream`).** *Upgraded from hedged assumption in first pass* ("heavy streaming logic … out of scope"). Now checked against `task/supervised.ex`:
  - `async_stream` options pass through `Task.Supervised.validate_stream_options/1` **before** any spawning (`task.ex:738`); that function hard-validates `max_concurrency` is an integer `> 0`, `on_timeout ∈ {:exit, :kill_task}`, and `timeout` is a non-negative integer or `:infinity`, raising `ArgumentError` otherwise (`supervised.ex:227-253`).
  - Concurrency is bounded: `stream_reduce` decrements `max` on each spawn and refuses to spawn when `max == 0`, only resuming after a task completes/dies (`supervised.ex:320-322,390-408`), so at most `max_concurrency` tasks run at once. Per-task timers are scheduled (`supervised.ex:571-575`) and cancelled on the task's `:DOWN` (`supervised.ex:601-603`); the supervisor's own `:max_children` is enforced (`supervised.ex:512-522,590-593`).
  - In `task.ex`, `await_many`/`yield_many` waits are bounded by the caller `timeout` (guarded by `is_timeout`); timer messages are cancelled and flushed in the `after`/terminal clauses (1003-1004, 1286-1287). **Confidence: high.** Basis: control-flow read of both files; no unbounded loop keyed on external input.

- **Deprecated `find/2` (line 1044).** `@doc false` + `@deprecated`; only pattern-matches messages the caller already holds, performs no dangerous operation. **Confidence: high.** Basis: read.

- **No sinks for command exec, shell, file/path handling, archive extraction, `binary_to_term` deserialization, `Regex.compile`, EEx, `Base`/`URI` round-trips, crypto comparisons, or externally-keyed global-state writes.** The only process-dictionary writes are `$ancestors`/`$callers`/`$initial_call` in `supervised.ex:83-93`, keyed on the spawning lineage (not external input). **Confidence: high.** Basis: grep (no matches) + full read of both files.

## Risks / caveats
- Reply-spoofing safety rests on the standard BEAM trust model: a task's alias `ref` is not deliberately published by the owner to untrusted processes. That is a caller responsibility this file cannot enforce.
- Verification ran against the in-repo build under a latin1 locale (harmless tokenizer warning on the test script, unrelated to results); behavioral outcomes (owner rejection, non-leaking `inspect`, arity-only `mfa`) are unaffected by encoding.

## Did I verify this?
- **Owner check** — yes, executed: non-owner `Task.await` → `ArgumentError`; owner → result. (`/tmp/t.exs`)
- **`inspect` non-leak** — yes, executed: struct inspect contained no secret; `mfa` held arity, not args.
- **MFA dispatch has no in-file untrusted transformation** — yes, verified by reading `task/supervised.ex:103-104,36,40,69-74` (resolves first-pass "not reviewed" hedge).
- **Stream option validation + concurrency bounding** — yes, verified by reading `task/supervised.ex:227-253,320-322,390-408,512-522,571-603` (upgrades first-pass "out of scope" assumption).
- **Reply-spoofing** — code paths verified by reading both files; the residual BEAM alias-unforgeability assumption is a platform property, not executed. Kept as a ruled-out with that caveat stated, not hidden behind "asserted."

### Dropped in re-review
None. Every prior ruled-out entry is preserved; two (MFA execution, resource consumption) were strengthened from asserted/out-of-scope to verified-by-reading against the neighbour module, and two (owner check, inspect leak) retain their executed-evidence basis re-confirmed this pass. No prior finding existed to drop, and no new finding rose above the low-confidence bar — the streaming path in `Task.Supervised` validates its inputs and bounds concurrency, so there was nothing to upgrade into a finding.
