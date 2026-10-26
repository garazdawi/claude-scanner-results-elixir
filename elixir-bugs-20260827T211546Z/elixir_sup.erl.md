# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_sup.erl
<!-- sha256: 3e7685b4be220e44e6142cc3bd1167563bbdf692374ab07bbec85412a88f48fb -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T06:26:21Z_

# Bug-hunting review — `elixir_sup.erl`

1. **Summary** — This is a textbook OTP supervisor with two permanent workers and a correctly-formed `one_for_one` strategy tuple; no bugs found.

2. **Bugfixes**

No findings

3. **Risks / caveats**

- The module has no `-spec` declarations and no non-builtin spec types, so the spec-totality checklist is vacuous — there are no exported functions with non-builtin spec types to answer entry-by-entry. The two exports are `start_link/0` and the `supervisor` behaviour callback `init/1`, both trivial.
- `init/1` matches only on the atom `ok`; that atom is the exact argument passed by `start_link/0` (`supervisor:start_link({local, ?MODULE}, ?MODULE, ok)`), the sole documented starter. A different `init` argument would `function_clause`, but no documented public path supplies one — `elixir_sup` is started by the OTP supervisor machinery via `start_link/0` (called by `elixir`, per the callgraph slice), never by an external caller passing a custom term.
- The child-spec tuples use the legacy 6-tuple form (`{Id, StartFunc, Restart, Shutdown, Type, Modules}`), which is still fully supported by `supervisor`. Restart type `permanent`, shutdown `2000`, type `worker`, and modules lists are all internally consistent with the actual modules named (`elixir_config`, `elixir_code_server`).
- The restart-strategy tuple `{one_for_one, 3, 10}` (intensity 3, period 10s) is well-formed and within the accepted ranges.

### Ruled out

**Operator / precedence typo**\
Reason: The file contains no guards, boolean expressions, or operators — only literal data structures and one function call.\
Axes examined: guard connectives, boolean ops, comparison ops | Not examined: none (no such constructs present in file).\
Basis: verified

**Missing clause / catch-all**\
Reason: `init/1` intentionally matches only `ok`, which is the only value `start_link/0` ever passes; no other documented caller exists.\
Axes examined: `ok` arg, non-`ok` atom, non-atom term | Not examined: none — only `start_link/0` reaches `init/1` and it passes the literal `ok`.\
Basis: verified

**Wrong atom / wrong symbol**\
Reason: Every atom (`elixir_config`, `elixir_code_server`, `permanent`, `worker`, `one_for_one`, `ok`) is a real, correct symbol for its position in the child spec / strategy tuple; module names match the `start_link` MFAs and modules lists.\
Axes examined: strategy atom, restart-type atom, worker-type atom, child module names | Not examined: none.\
Basis: verified

**State / accumulator threading error**\
Reason: No state threading, accumulators, or rebinding — `init/1` builds a constant list and returns it.\
Axes examined: variable rebind, accumulator reuse | Not examined: none (no variables threaded).\
Basis: verified

**Off-by-one / wrong arithmetic**\
Reason: The only integers are the shutdown timeout (`2000`), restart intensity (`3`), and period (`10`); all are plain literals with no arithmetic and are valid supervisor parameters.\
Axes examined: shutdown value, intensity, period | Not examined: none.\
Basis: verified

**Contract violation**\
Reason: `init/1` returns the documented `{ok, {SupFlags, [ChildSpec]}}` shape exactly, using the accepted legacy tuple forms for both flags and children.\
Axes examined: return-tuple shape, child-spec arity, sup-flags arity | Not examined: none.\
Basis: verified

**Crash on stray / unexpected input**\
Reason: The only input is `init/1`'s argument, fixed to `ok` by the sole caller; no external/untrusted data reaches this module.\
Axes examined: `init` arg value | Not examined: none.\
Basis: verified

**Resource leak on error path**\
Reason: The module allocates no resources; child processes are owned and cleaned up by the OTP supervisor, not by this code.\
Axes examined: file/ETS/process/monitor creation | Not examined: none (none created here).\
Basis: verified

**Silent corruption**\
Reason: The function returns a static, correct configuration structure; there is no data transformation that could silently produce a wrong-but-plausible value.\
Axes examined: returned structure content | Not examined: none.\
Basis: verified

**Infinite loop / hang**\
Reason: No recursion, no `receive`, no loops — `init/1` returns immediately.\
Axes examined: recursion, receive, loop | Not examined: none.\
Basis: verified

**Quadratic-where-linear**\
Reason: No iteration or list accumulation; the worker list is a fixed 2-element literal.\
Axes examined: list building | Not examined: none.\
Basis: verified

**Repeated work**\
Reason: `init/1` runs once at supervisor start and does no per-call recomputation of patterns or scans.\
Axes examined: per-call recompute | Not examined: none.\
Basis: verified

**Dead / unreachable code**\
Reason: Both exports are reachable: `start_link/0` from module `elixir` (per callgraph slice) and `init/1` from the `supervisor` behaviour via that `start_link/0`.\
Axes examined: export reachability | Not examined: none.\
Basis: verified

**Documented behaviour mismatch**\
Reason: No docstrings or `-spec`s to contradict; the inline comments accurately describe the legal value ranges for each child-spec field and match the literals used.\
Axes examined: comment-vs-code for restart/shutdown/type/modules | Not examined: none.\
Basis: verified

**Race condition / shared mutable state**\
Reason: `init/1` is pure and touches no shared state; process/table ownership and concurrency are handled by the supervised gen_servers, not here.\
Axes examined: shared-state access | Not examined: none (none present).\
Basis: verified

**Wrong scope / order of operations**\
Reason: There is a single expression with no side effects and no ordering dependency.\
Axes examined: side-effect ordering | Not examined: none.\
Basis: verified

**Encoding / unicode mishandling**\
Reason: No string, binary, or chardata handling anywhere in the file.\
Axes examined: string/binary handling | Not examined: none (none present).\
Basis: verified

### Did I verify this?

Every ruled-out entry above is marked `verified`, and each is legitimately so: the file is 36 lines of pure supervisor boilerplate with no guards, no recursion, no variables threaded, no strings/binaries, no allocations, and a single constant return value. For each class the `Not examined` list reads `none` because the datatype axes the class targets are simply not instantiated by this file — I read the whole file (all 36 lines) and confirmed the absence of each construct rather than inferring it. No `asserted` dismissals remain, so no `Depends on:` lines are required. The one cross-file assumption — that `elixir_sup` is started only via `start_link/0` with the literal `ok`, so `init/1`'s single clause never faces a stray argument — rests on the provided callgraph slice (called by `elixir`, calls into nothing) and OTP's own `supervisor:start_link/3` contract, which forwards exactly the third argument to `init/1`.

