# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/error_handler.ex
<!-- sha256: 8a001c513d47d859f60b4a0c8c9d8dc3b0a17156af944ec255f2fb839093b2f3 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/error_handler.ex
<!-- sha256: 8a001c513d47d859f60b4a0c8c9d8dc3b0a17156af944ec255f2fb839093b2f3 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T (neighbour modules read)_

I re-read the entire 60-line file and, for each hedge in the first pass, read the callgraph neighbour that the hedge deferred to (`parallel_compiler.ex`). Below is the report.

---

## Summary

`Kernel.ErrorHandler` is a small internal compiler-coordination module (`@moduledoc false`) invoked by the BEAM runtime as an installed `error_handler` and by `Kernel.ParallelCompiler`. Its functions receive already-interned atoms, unforgeable refs, and trusted compiler pids. No path converts untrusted bytes into a dangerous sink. The three items the first pass ruled out on assumption are now **verified against the neighbour module** that owns the other end of the send/receive protocol; they remain ruled out, this time on read evidence rather than assertion. No security findings.

## Security

`No findings`

Complete set of outbound calls: `:code.ensure_loaded/1`, `:error_handler.undefined_function/3` & `undefined_lambda/3`, `:erlang.get/1`, `:erlang.make_ref/0`, `:elixir_module.compiler_modules/0`, `send/2`, `:erlang.garbage_collect/1`, `:erlang.monitor/2`, and two `receive` blocks. None of the atom-creation, command, file, deserialization, template, crypto, or Unicode sink shapes are present. The four items worth stating explicitly are ruled out below.

### Ruled out

- **Atom-table exhaustion (CWE-770).** Confidence: high. No `String.to_atom/1`, `binary_to_atom`, `Module.concat/1`, or any bytes→atom conversion exists anywhere in the file. `module` and `fun` arrive as pre-existing atoms (the runtime invokes the installed error handler with the atoms of the undefined call, interned earlier by the tokenizer/parser). Basis: complete read of the 60-line file — exhaustive enumeration over the whole search space, not a partial scan. Axes examined: bytes→atom conversion presence, argument provenance. Not examined: none.

- **Arbitrary code execution via `:error_handler.undefined_function/undefined_lambda`.** Confidence: high. These dispatch the *original* call (`module.fun(args)`) after ensuring the target is loaded/compiled; they introduce no new callable beyond the one the failing call already named. `:code.ensure_loaded/1` only loads a module already on the trusted code path and does not invoke functions. Basis: full-file dataflow (line 12→13, 18→19) with no transformation of the target. Axes examined: target substitution, new-callable introduction, loader side effects. Not examined: none.

- **Process-dictionary read `:erlang.get(:elixir_compiler_info)` (line 37) crashing on absence.** Confidence: high. **Verified against neighbour.** If the key were unset, the tuple match at line 37 would raise `MatchError`. I read `parallel_compiler.ex:543-546`: every compilation worker is spawned via `:erlang.spawn_monitor`, and the very first statements in the worker body seed the key — `:erlang.put(:elixir_compiler_info, {parent, self()})` at line 545 — **before** any `compile_file`/`require_file` runs (lines 548-553). The error handler is only installed on a process while it is compiling, i.e. inside that worker body, so the key is always present when `ensure_compiled/4` reaches line 37. This is a trusted-setup invariant enforced by the parent compiler, not attacker-influenced state. Basis: read of `parallel_compiler.ex:537-560`. Axes examined: key-seeding site, ordering vs. compilation, reachability outside a compiling process. Not examined: none.

- **Unbounded `receive` hang / attacker-forgeable coordination messages (resource consumption + concurrency).** Confidence: high (for the source-attacker threat model). **Both ends now verified.**
  - *Forgeability:* Both `receive` blocks (lines 48-58, 52-54) match on a fresh `:erlang.make_ref/0` value (outer) or a monitor ref (inner), both process-local and unforgeable — untrusted source being compiled cannot inject a matching message into the mailbox.
  - *Inner receive (lines 52-54) cannot hang:* the monitored `pid` is the loader from `parallel_compiler.ex:940-962` (`load_module`), a short-lived `spawn_opt` process whose entire body is one `:code.load_binary/3` call followed by exit. The monitor therefore always receives `:DOWN` in bounded time. Executed check (prior pass, still valid): `erl -noshell -eval 'Pid = spawn(fun() -> ok end), timer:sleep(50), Ref = erlang:monitor(process, Pid), receive {'DOWN',Ref,_,_,R} -> ... after 2000 -> HANG end'` printed `DOWN delivered immediately, reason=noproc` — a monitor on an already-dead loader delivers `:DOWN` immediately.
  - *Outer receive (lines 48-58) liveness:* I read the compiler side of the `:waiting` protocol in `parallel_compiler.ex:797-831` and the deadlock resolver at `615-647`, `712-725`. For every `{:waiting, …}` message the compiler replies to the child with exactly one of `{ref, {:loading, pid}}`, `{ref, :found}`, `{ref, :not_found}`, or `{ref, :deadlock}` — including the deadlock-resolution path (`deadlocked/without_definition` → `{pid, :deadlock}`/`{pid, :not_found}`), which exists precisely so a waiting child is never left blocked. So while `compiler_pid` is alive, the outer receive always gets a reply.
  - Basis: read of `parallel_compiler.ex:766-831, 915-962`, plus the executed monitor check. Axes examined: ref forgeability, inner-monitor liveness (executed + loader-body read), outer-reply guarantee under the deadlock protocol (neighbour read). **Not examined:** `compiler_pid` (the parent `ParallelCompiler` process) crashing mid-wait — the worker is `spawn_monitor`'d by the parent but is *not* linked back to it, so a parent crash would leave the outer `receive` (which has no `after`) blocked forever. This axis is not source-attacker-reachable: `compiler_pid` is the trusted compile driver, and per the folder threat model a crash of a trusted supervising process is explicitly out of scope (an attacker supplying untrusted source cannot induce it). I did not build a full parallel-compilation harness to force a parent crash, so this axis stays untried; it does not change the source-attacker verdict.

## Did I verify this

- **Verified by reading neighbours (this pass):** the `:elixir_compiler_info` seeding invariant (`parallel_compiler.ex:543-546`), the always-reply guarantee of the `:waiting`/deadlock protocol (`797-831`, `615-647`, `712-725`), and the bounded lifetime of the monitored `:loading` loader pid (`940-962`). These three were `asserted`/"out of scope" in the first pass and are now backed by neighbour reads. The first-pass hedges "depends on … which live in other files and were out of scope" and "single-file review" are retired accordingly.
- **Verified by execution:** the inner-receive no-hang behaviour (monitor on a dead process → immediate `:DOWN`), via the `erl` one-liner above.
- **Still untried (honestly noted, not cleared):** whether a crash of the trusted parent `compiler_pid` mid-wait hangs the outer timeout-less `receive`. Reading the neighbour tells me the worker is monitored-but-not-linked; it does not let me *clear* the hang axis, which would require forcing the parent to die under a live compile. This axis is out of the source-attacker threat model, so the entry stays ruled out rather than becoming a finding.

## Dropped in re-review

None. Every prior ruled-out entry is preserved; three were upgraded from `asserted` to `verified` against `parallel_compiler.ex`, and one (the `receive` entry) retains one explicitly-labelled untried axis (`compiler_pid` death) that remains out of scope for the source-supplier attacker. No prior finding existed to drop, and no new finding was warranted — the only residual axis is not attacker-reachable, so upgrading it to a low-confidence finding would be unsupported.
