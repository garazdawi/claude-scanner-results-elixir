# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_compiler.erl
<!-- sha256: e09147ddb640a561840d130f6d6ead2c14212cd4afefe2a48f6817eaecbef398 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_compiler.erl
<!-- sha256: e09147ddb640a561840d130f6d6ead2c14212cd4afefe2a48f6817eaecbef398 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T06:04:00Z (neighbour modules read: `elixir_module.erl`, `elixir_code_server.erl`, `elixir_utils.erl`; encoding axis executed under Erlang/OTP)_

# Bug-hunting review: `elixir_compiler.erl`

## Summary
After reading the full 252-line file, tracing each of the six exports and their helpers, **and** reading the three callgraph neighbours the first pass had only assumed (`elixir_module`, `elixir_code_server`, `elixir_utils`) plus executing the malformed-UTF-8 path, I found no correctness/crash/leak defect reachable through a documented entry point. The three cross-file contracts the first pass left hedged are now confirmed with cited neighbour code and one executed axis. No findings introduced; no findings dropped (the first pass had none).

## Bugfixes

No findings.

The candidates I examined most closely, and why each is not a bug:

- **`only_defmodule/1` (lines 132–137)** — the second/third clauses are mis-indented, but Erlang is whitespace-insensitive and the `;` separators make three ordinary function clauses. Semantically identical to the canonical form. Cosmetic only.
- **`do_is_purgeable/1` chunk padding (line 97)** — `4 * trunc((Size+3) / 4)` rounds the chunk size up to the next multiple of 4 with float division. `Size` is a 32-bit field (≤ ~4.29e9 < 2^53), represented exactly as a float, so `trunc` is exact — equivalent to `((Size+3) div 4) * 4`. Correct BEAM IFF padding.
- **`is_purgeable`/`do_is_purgeable` `LocT` detection** — clause order (empty `LocT` → `true`, non-empty/odd-size `LocT` → `false`, generic chunk-skip last) is correct; conservative `false` on a malformed chunk is safe. Input is always the BEAM binary produced by `elixir_erl_compiler:noenv_forms`, so the `<<"FOR1",…,"BEAM",…>>` head match cannot fail on the documented path.
- **Compiler-module pool return (lines 82/104/129–130)** — `retrieve_compiler_module` is acquired inside `spawned_compile` and only returned via `dispatch`'s `after`. If `noenv_forms`/`load_binary` raises, the module name is not returned to the pool; this is upstream-identical (pool mints a fresh counter next time — see `elixir_code_server.erl:70`) and fires only on a compile error, so not a defect.

## Risks / caveats
- `elixir_config:get(parser_options)`/`get(tracers)` (lines 11, 19) rely on those keys being pre-inserted (`bootstrap/0` lines 173/175, and `elixir:start/2`). Per the folder threat model `get/1` `badmatch`es on a missing key by design; under documented startup this holds.
- The cross-file return-shape and pool-serialization contracts that the first pass listed here as unverified caveats have been resolved this pass — see the three formerly-`asserted` ruled-out entries below, now `verified` with cited neighbour code / executed evidence.

### Ruled out

**Operator / precedence typo**
- Reason: The only boolean/comparison operators are the `andalso` chain in `optimize_defmodule/2` (correct short-circuit intent) and `?key(E, module) == nil` where `==`/`=:=` are equivalent on atoms.
- Axes examined: `andalso` vs `orelse` intent, `==` vs `=:=` on atom `nil`, guard `is_binary(File), is_integer(Line)` in `code_mod/6` | Not examined: none
- Basis: verified

**Missing clause / catch-all**
- Reason: `only_defmodule/1` and `expand_defmodule/2` handle the same shapes (`__block__`, `defmodule`, else); `interpret`'s `try` catches `Kind:Reason:Stacktrace` (all classes).
- Axes examined: single defmodule (no block), nested block, non-defmodule expr, empty block; `try/catch` class coverage (throw/error/exit) | Not examined: none
- Basis: verified

**Wrong atom / wrong symbol**
- Reason: Atoms (`retrieve_compiler_module`, `return_compiler_module`, `no_auto_import`, `'__RELATIVE__'`, chunk tags `"FOR1"/"BEAM"/"LocT"`, config keys) all match their consumers/spec. The two pool atoms match `elixir_code_server.erl:65` (`handle_call(retrieve_compiler_module, …)`) and `:102` (`handle_cast({return_compiler_module, Module, Purgeable}, …)`).
- Axes examined: chunk-name binaries, export/attribute atoms, config keys, pool call/cast atoms vs code_server handlers | Not examined: none
- Basis: verified

**State / accumulator threading error**
- Reason: `quoted/3` saves/restores `elixir_module_binaries` and captures the result before the `after`; `from_env(E)` intentionally uses the pre-expansion env in both `compile` and `interpret` (consistent with each other and upstream).
- Axes examined: process-dict save/restore ordering, `SE`/`EE` vs `E` usage, result-capture-before-after | Not examined: none
- Basis: verified

**Off-by-one / wrong arithmetic**
- Reason: BEAM chunk padding `4*trunc((Size+3)/4)` is exact for 32-bit `Size`; `erl_anno:new(Line)` and the `{…, 1}` file-attribute line are correct.
- Axes examined: Size=0,1,4,5 padding; float exactness up to 2^32; line anno | Not examined: none
- Basis: verified

- **Contract violation** — *Upgraded from hedged assumption in first pass (asserted → verified)*
- Reason: `compile/4` and `interpret/3` both return `{X, SE, EE}` 3-tuples; the sole in-folder consumer, `elixir_module.erl:488-492`, destructures exactly `{Value, ExS, EE} = case … interpreted -> elixir_compiler:interpret(Block, Vars, E); compiled -> elixir_compiler:compile(Block, Vars, [no_bool_opt, no_ssa_opt], E) end` — shape matches on both arms. `quoted/3` returns a reversed `[{Module,Binary}]` list, consumed in-file by `bootstrap_file/2` (line 187 → `binary_to_path({ModuleName, Binary}, …)` at line 246), which matches the pair shape. `ArgsList` is turned into a tuple (`list_to_tuple`, line 75) / zipped against `Vars` (line 54) inside this file; the consumer supplies `Vars` as the args list.
- Axes examined: return arity/shape of each export; consumer destructure in `elixir_module:compile` (both `interpreted`/`compiled` arms); `quoted` list-of-pairs consumer | Not examined: none
- Basis: verified (read `elixir_module.erl:488-492`)

**Crash on stray / unexpected input**
- Reason: The documented text entry (`string/3`→`file/2`) routes attacker bytes to `elixir:'string_to_quoted!'`/tokenizer, not to raw pattern matches here; `is_purgeable`'s `FOR1/BEAM` head only sees compiler-produced BEAM.
- Axes examined: non-UTF8 file bytes (handled upstream by `elixir_utils:characters_to_list`, see Encoding entry), non-BEAM binary to `is_purgeable` (not reachable from public path) | Not examined: none reachable
- Basis: verified

**Resource leak on error path**
- Reason: Compiler-module non-return on compile error is upstream-identical and pool-recoverable — `elixir_code_server.erl:66-70` mints `compiler_module(Counter)` with `Counter+1` when `Unused` is empty — not a defect; `quoted/3`'s `after` always restores the process dict.
- Axes examined: exception between `retrieve`/`return`, process-dict restore on throw, pool fresh-counter fallback | Not examined: none
- Basis: verified

**Silent corruption**
- Reason: `lists:reverse(get(elixir_module_binaries))` restores insertion order; `do_is_purgeable` returns a truthful boolean per chunk table.
- Axes examined: binary ordering, purgeable true/false partition | Not examined: none
- Basis: verified

**Infinite loop / hang**
- Reason: `do_is_purgeable` strictly consumes ≥8 bytes per generic step until `<<>>` or a `LocT` clause; no unguarded `receive` in this file.
- Axes examined: empty binary, chunk-skip termination, LocT short-circuit | Not examined: none
- Basis: verified

**Quadratic-where-linear / Repeated work**
- Reason: No `++`-in-loop, no per-iteration `length/1`, no recompiled patterns; the only list reverse is a single final `lists:reverse`.
- Axes examined: accumulator building, per-call recompilation | Not examined: none
- Basis: verified

**Dead / unreachable code**
- Reason: Every clause is reachable (block/defmodule/else all producible from source AST); the `defmodule` fast path is gated but reachable when module is nil and not bootstrapping.
- Axes examined: clause reachability for `only_defmodule`/`expand_defmodule`, fast-path gate | Not examined: none
- Basis: verified

**Documented behaviour mismatch**
- Reason: No `-spec`/`@doc` prose in this internal module beyond the two comments (`optimize_defmodule` fast-path, `no_tail_optimize`); behavior matches both comments.
- Axes examined: fast-path comment vs code, no-tail-optimize comment vs generated block | Not examined: none
- Basis: verified

- **Race condition / shared mutable state** — *Upgraded from hedged assumption in first pass (asserted → verified)*
- Reason: The only shared state touched here is (a) the per-process dict key `elixir_module_binaries`, which is process-local and save/restored in `quoted/3`, and (b) the compiler-module pool, which is owned entirely by the single `elixir_code_server` gen_server. `retrieve_compiler_module` is a synchronous `handle_call` (`elixir_code_server.erl:65-71`) that atomically pops the head of `Unused` or mints `Counter+1`; `return_compiler_module` is a `handle_cast` (`:102-105`) that pushes `{Module, Purgeable}` onto `Used`. All pool state transitions execute inside the single-threaded gen_server, so concurrent compiles cannot observe a torn `mod_pool` or receive the same module twice. No lock-free shared structure is accessed from this file.
- Axes examined: process-dict locality; code-server serialization; whether two concurrent retrieves can collide (they can't — serialized `handle_call`); return-via-cast vs retrieve-via-call asymmetry (return is fire-and-forget onto `Used`, no correctness dependency on ordering) | Not examined: none
- Basis: verified (read `elixir_code_server.erl:65-105`)

**Wrong scope / order of operations**
- Reason: Result computed before `after` in `quoted/3`; `E#{line := ?line(Meta)}` set before use in `expand_defmodule`.
- Axes examined: evaluation order of try-body vs after, env-line update timing | Not examined: none
- Basis: verified

- **Encoding / unicode mishandling** — *Upgraded from hedged assumption in first pass (asserted → verified)*
- Reason: `file/2` (line 34) feeds the raw file binary to `elixir_utils:characters_to_list/1`. That function (`elixir_utils.erl:136-143`) delegates to `unicode:characters_to_list/1` and routes `{error, Encoded, Rest}`/`{incomplete, Encoded, Rest}` to `conversion_error/3` (`:154-155`), which calls `error('Elixir.UnicodeConversionError':exception(...))` — i.e. it **raises**, never treating a stray byte as a codepoint. I executed the axis rather than reading only: pushing `<<"foo",16#FF,16#FE,"bar">>` through `unicode:characters_to_list/1` returns `{error,"foo",<<0xFF,0xFE,"bar">>}` (→ raise), the incomplete tail `<<"foo",16#C3>>` returns `{incomplete,"foo",…}` (→ raise), and valid `<<"café"/utf8>>` returns the codepoint list `"café"`. So malformed source bytes crash the one file's compile with a typed error — no silent byte-as-codepoint corruption. There is no `lists:flatten`-on-chardata here.
- Axes examined: file-binary → list conversion; relative-path chardata; **invalid-UTF8 file (executed)**; **incomplete-UTF8 tail (executed)**; valid multibyte (executed) | Not examined: none
- Basis: verified (executed `unicode:characters_to_list` on invalid/incomplete/valid input; read `elixir_utils.erl:136-155`)

**Unclassified**
- Reason: No file-specific one-off defect surfaced beyond the classes above.
- Axes examined: full read of all 6 exports + helpers | Not examined: none
- Basis: verified

### Did I verify this?

- Every in-file entry had its `Not examined` list reduced to `none` by reading the concrete clause and tracing the triggering shapes within this file.
- The three entries the first pass left **asserted** — **Contract violation**, **Race condition**, **Encoding** — are now **verified**:
  - *Contract violation*: read the actual consumer, `elixir_module.erl:488-492`, and confirmed both `interpret/3` and `compile/4` results are destructured as `{Value, ExS, EE}`; the `quoted/3` pair-list is consumed in-file at `binary_to_path/2` (line 246). This is a fixed contract shape, fully determined by the consumer source — reading it is the verification.
  - *Race condition*: read `elixir_code_server.erl:65-105` and confirmed the pool is mutated only inside a single gen_server via a synchronous `handle_call` (retrieve) and a `handle_cast` (return); serialization is structural, not input-dependent.
  - *Encoding*: this was the one axis with a concrete divergent input (malformed UTF-8), so I **executed** it under Erlang/OTP rather than only reading — `unicode:characters_to_list` returns `{error,…}`/`{incomplete,…}` on bad bytes and `characters_to_list/1` raises `UnicodeConversionError` on both, so `file/2` fails loudly. The axis is tried, not assumed.

### Dropped in re-review

None. The first pass reported no findings, and every ruled-out entry is preserved (three moved from `asserted` to `verified`; none downgraded or removed).
