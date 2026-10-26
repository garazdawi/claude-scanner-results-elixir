# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_utils.erl
<!-- sha256: 287f47e07464c121a53318a360faec0309bb59172d9b150986fa5b5f4d20535b -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_utils.erl
<!-- sha256: 287f47e07464c121a53318a360faec0309bb59172d9b150986fa5b5f4d20535b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T (hedge-resolution pass)_

I re-read the full file and the in-folder callgraph neighbour `elixir_config.erl` (the only mutation-relevant callee reachable from this module besides `elixir_aliases`). Below, the two first-pass `asserted` ruled-out entries are each resolved: one upgraded to `verified` by reading the neighbour's source, one kept `asserted` because its untried axis cannot be executed and its dependency is an out-of-folder OTP module.

## Summary
`elixir_utils.erl` is largely clean; the one real defect is that `get_line/2` and `get_file/2` crash with `case_clause` (rather than falling back gracefully, as their sibling `get_line/1` does) when a `line`/`file` entry is present but of the wrong type, reachable from the documented `Code.eval_*` opts path.

## Bugfixes

#### `CSCAN-elixir_utils-get-line-file-wrong-type-case-clause`
- **Finding ID:** CSCAN-elixir_utils-get-line-file-wrong-type-case-clause
- **Severity:** Low
- **Confidence:** medium
- **Location:** `elixir_utils.erl:175-179` (`get_line/2`) and `elixir_utils.erl:181-185` (`get_file/2`)
- **Class:** Crash on stray / unexpected input (asymmetric with the graceful `get_line/1` at lines 169-173)
- **User-visible outcome:** A caller of `Code.eval_string/3`, `Code.eval_quoted/3`, or `Code.env_for_eval/1` who passes `line:` as a non-integer (e.g. `line: "5"`) or `file:` as a non-binary reaches `env_for_eval/1` (`elixir.erl:223-224`), which calls `get_line(Opts, Env)` / `get_file(Opts, Env)`. `lists:keyfind` returns e.g. `{line, "5"}`; the `is_integer` guard on the first clause (line 177) fails and the only other clause is `false ->` (line 178), which does not match the `{line, "5"}` tuple, so evaluation aborts with a bare `case_clause` error. The caller observes an opaque `{case_clause,{line,"5"}}` crash from deep inside the compiler instead of either the graceful fallback that `get_line/1` gives (`_ -> 0`, line 172) or a clean `ArgumentError`. Without this bug the same malformed option would either be ignored (fall back to the env's line/file) or reported cleanly.
- **Reproduction:** `Code.eval_string("1 + 1", [], line: "5")` (or `file: 'charlist'`). A naïve attempt with `line: 5` (correct integer) succeeds; the crash needs the wrong-typed value. The identical shape exists for `module:`/`tracers:` in `elixir.erl` itself, out of this file's scope.
- **Suggested fix:** Change the fall-through clause in both functions from `false ->` to `_ ->`, matching `get_line/1`: `get_line/2` line 178 → `_ -> ?key(Env, line)`; `get_file/2` line 184 → `_ -> ?key(Env, file)`.
- **Risk of fix:** none — the change only converts a crash into the already-intended fallback; well-formed opts are unaffected.
- **Evidence:**
  - read `elixir_utils.erl:169-185` (the `_ ->` vs `false ->` asymmetry)
  - read `elixir.erl:221-240` (`env_for_eval/1` passes user `Opts` straight to `get_line/2`/`get_file/2`)
  - relies on Code module docs: `:line` documented as integer, `:file` as binary (contract caveat below)

## Risks / caveats
- The finding above is borderline: the `:line`/`:file` options are documented to be an integer / binary, so a wrong-typed value is arguably a caller contract violation (which this review is told to de-prioritise). I report it because the sibling `get_line/1` deliberately degrades gracefully (`_ -> 0`) on malformed metadata, establishing that graceful tolerance is the module's own convention, and because the crash surfaces as an unhelpful internal `case_clause` rather than a clean error. Left to the validator to weigh.
- Reachability of the `env_for_eval/1` path was confirmed by reading `elixir.erl`; I did not execute the code (no `ran` evidence), so the exact crash term is inferred from the clause structure, not observed. Confidence stays medium for this reason.
- The empty-`__block__` concern in `returns_boolean` (`lists:last([])`) is listed under Ruled out; I could not construct a path that feeds `{'__block__', _, []}` into it, and it is faithful to upstream, so it is not reported as a finding.

### Ruled out
- **Operator / precedence typo**
  - Reason: Guards in `returns_boolean/1` and `var_info/2` use `;`/`==` consistently with intent; every comparison/boolean op atom checked returns boolean, and `andalso`/`orelse` correctly recurse only on the right operand.
  - Axes examined: all 8 comparison ops, `and`/`or`/`xor`, unary type-checks, 2/3-arg BIFs, `andalso`/`orelse` short-circuit semantics | Not examined: none
  - Basis: verified

- **Missing clause / catch-all**
  - Reason: `guard_op/2` catches `_:_` from `erl_internal:op_type/2` and covers all five documented return atoms; `relative_to_cwd/1` catches `_:_`; the `case`/`cond` funs in `returns_boolean` match the only valid `{'->',...}` clause shape.
  - Axes examined: non-operator atom to `guard_op`, unknown `op_type` return, config-lookup failure in `relative_to_cwd` | Not examined: `op_type` returning a novel atom (would escape `of` as `try_clause`, but domain is fixed to 5 values)
  - Basis: verified

- **Wrong atom / wrong symbol**
  - Reason: `erlang_comparison_op_to_elixir/1` maps `/=,=<,=:=,=/=` correctly with an `Other` fallback; `macro_name`/`erl_fa_to_elixir_fa` use the literal `"MACRO-"` prefix consistently; all BIF/record-field atoms in `returns_boolean` exist.
  - Axes examined: each op-atom mapping, `MACRO-` prefix round-trip, every `is_*`/`function_exported`/`is_record` atom | Not examined: none
  - Basis: verified

- **State / accumulator threading error**
  - Reason: `split_last/2` accumulates with `[H|Acc]` and reverses once; `extract_or_guards`/`extract_splat_guards` thread correctly; no stale-var reuse.
  - Axes examined: empty list, single element, multi element through `split_last`/`split_opts`/`extract_splat_guards` | Not examined: improper list (spec forbids)
  - Basis: verified

- **Off-by-one / wrong arithmetic**
  - Reason: only arithmetic is `Arity - 1` in `erl_fa_to_elixir_fa/2` for the `MACRO-` case, which is the correct macro-arity adjustment.
  - Axes examined: macro vs non-macro name, arity value | Not examined: none
  - Basis: verified

- **Contract violation (return shape)**
  - Reason: `read_file_type`/`read_link_type`/`read_posix_mtime_and_size` (lines 99-118) match only the documented `{ok, #file_info{...}}`/`{error, _}` shapes from `file:read_file_info/read_link_info`; conversion helpers raise the documented `UnicodeConversionError`.
  - Axes examined: ok path, error path for each file function (both arms are pattern-matched in-source, lines 104-105, 110-111, 116-117) | Not examined: file functions returning a third shape other than `{ok,_}`/`{error,_}` — this would `case_clause` at lines 104/110/116.
  - Basis: **asserted** — the sole untried axis (a non-`{ok,_}`/`{error,_}` return from `file:*`) is **un-triable here**: `kernel/file` is an OTP module, not an in-folder callgraph neighbour (this module's in-folder callees are only `elixir_aliases` and `elixir_config`), and a third return shape cannot be forced by execution, so per the re-review rules the axis stays on `Not examined` and the entry stays `asserted`. Reading the OTP source would not clear the axis either. Kept asserted rather than upgraded to a finding because the OTP `file` return contract is a documented spec guarantee and upgrading would be a false positive.
  - Depends on: `kernel/file` — `read_file_info/2`/`read_link_info/1`/`write_file_info/3` return only `{ok,#file_info{}}` or `{error,_}`.

- **Crash on stray / unexpected input** — became the finding above (`get_line/2`, `get_file/2`).

- **Resource leak on error path**
  - Reason: no files/ETS/processes/monitors are opened or held in this module; file functions are one-shot stat/write calls with no handles retained.
  - Axes examined: every file:*/config call site | Not examined: none
  - Basis: verified

- **Silent corruption**
  - Reason: `split_last([])` → `{[], []}` is an intentional documented special case; `generated/1` prepending a duplicate `{generated,true}` is harmless (keyfind takes the first); no value is silently truncated or mis-returned.
  - Axes examined: empty/single/multi list for `split_last`; generated-first vs generated-absent vs generated-not-first | Not examined: none
  - Basis: verified

- **Infinite loop / hang**
  - Reason: all recursion (`split_last`, `extract_or_guards`, `returns_boolean` on `__block__`) strictly consumes list structure; no `receive`.
  - Axes examined: recursion decreases on list length | Not examined: `returns_boolean` on `{'__block__',_,[]}` → `lists:last([])` raises (crash, not hang; see caveat)
  - Basis: verified

- **Quadratic-where-linear**
  - Reason: `split_last/2` uses cons+single-reverse; `InnerOpts ++ OuterOpts` in `split_opts` is a single bounded append, not in a loop.
  - Axes examined: accumulator construction, single vs looped `++` | Not examined: none
  - Basis: verified

- **Repeated work**
  - Reason: no compiled patterns / regex / BEAM scans; helpers are simple keyfinds.
  - Axes examined: per-call recompilation sites | Not examined: none
  - Basis: verified

- **Dead / unreachable code**
  - Reason: every clause is reachable from an exported function; the `send`/`list` arms of `guard_op` are reachable for `!`/`++`/`--` operators.
  - Axes examined: each `returns_boolean`/`guard_op` clause | Not examined: none
  - Basis: verified

- **Documented behaviour mismatch**
  - Reason: `meta_keep/1` returns `{binary,integer}` or `nil` exactly as its comment states; `noop/0` returns `ok` as documented; `split_last`'s empty special-case is called out in `split_opts`'s usage.
  - Axes examined: `meta_keep` with valid pair / absent / wrong-typed pair; `noop` | Not examined: none
  - Basis: verified

- **Race condition / shared mutable state** — **Upgraded from hedged assumption in first pass** (asserted → verified)
  - Reason: the only mutation-relevant cross-module call in this module is `elixir_config:get(relative_paths)` at `elixir_utils.erl:129` (via `relative_to_cwd/1`). Reading the in-folder neighbour `elixir_config.erl:24-26` confirms `get/1` is `[{_, Value}] = ets:lookup(?MODULE, Key), Value` — a pure ETS read with `read_concurrency` (table created `{read_concurrency, true}` at `elixir_config.erl:46`); it performs no `ets:insert`, no `persistent_term:put`, and no `gen_server:call`. All writes in `elixir_config` (`put`/`update`/`get_and_put`, lines 36-43) go through the gen_server and are not on this module's call path. So no shared-state write is reachable from `elixir_utils`. `elixir_aliases:inspect/1` (line 22) is read-only formatting. `Not examined` was already `none`; the only reason the first pass left this `asserted` was the unread dependency, which I have now read directly.
  - Axes examined: all cross-module calls (`elixir_config:get/1` read path, `elixir_aliases:inspect/1`) | Not examined: none
  - Basis: **verified** (by reading `elixir_config.erl:24-26,36-48` — an in-folder callgraph neighbour; no differential axis is involved, so source inspection is conclusive)

- **Wrong scope / order of operations**
  - Reason: no side-effect-before-validation; file writes occur after guard validation of args (`change_posix_time`/`change_universal_time` guard `is_integer` before `file:write_file_info`).
  - Axes examined: guard-before-write ordering | Not examined: none
  - Basis: verified

- **Encoding / unicode mishandling**
  - Reason: `characters_to_list`/`characters_to_binary` handle `{error,_}`/`{incomplete,_}` from the `unicode` module and raise the proper exception; list/binary input is passed through per contract.
  - Axes examined: binary input, list passthrough, error tuple, incomplete tuple | Not examined: improper/nested chardata (delegated to `unicode` module, spec-permitted)
  - Basis: verified

### Did I verify this?
- Classes marked **verified** above: I read the relevant clauses and traced each listed axis through the code; their `Not examined` lists read `none` (or name only spec-forbidden/other-module inputs that are out of scope), so they stand as verified.
- **Race condition / shared mutable state** — **upgraded to verified** this pass by reading the in-folder neighbour `elixir_config.erl`. `get/1` (lines 24-26) is a bare `ets:lookup`; no write is reachable from `relative_to_cwd/1`. Because the entry had no untried differential axis (`Not examined: none`) and the sole reason for the prior `asserted` was the unread dependency, direct source inspection of the neighbour is conclusive here. `Depends on:` line dropped as the dependency is now confirmed in-repo.
- **Contract violation (return shape)** — **kept asserted.** Its one untried axis ("`file:*` returns a shape other than `{ok,_}`/`{error,_}`") cannot be exercised: `kernel/file` is not an in-folder callgraph neighbour, and no execution can force OTP's `file` module to emit a third shape. Reading OTP source would not clear the axis (reading tells what the code does, not which inputs disagree), so per the re-review rules the axis stays on `Not examined` and the entry stays `asserted`. Not upgraded to a finding because the OTP return contract is a documented guarantee and a finding would be a false positive. `Depends on:` line retained.
- The reported finding (`get_line/2`/`get_file/2`) rests on `elixir.erl:221-224` feeding user-controlled `Opts` into these functions; I read that caller directly. I did not execute the crash, so confidence is medium rather than high, and the contract caveat is disclosed above.

### Dropped in re-review
- No findings dropped. The single first-pass finding (`CSCAN-elixir_utils-get-line-file-wrong-type-case-clause`) is preserved unchanged; both flagged `asserted` ruled-out entries were resolved (one upgraded to `verified` via neighbour read, one retained as `asserted` with an un-triable axis).
