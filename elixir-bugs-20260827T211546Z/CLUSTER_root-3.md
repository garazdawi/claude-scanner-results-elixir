# Cluster cross-file findings

#### `CSCAN-cluster-map-repeated-key-passes-file-not-env`

- **Finding ID** — CSCAN-cluster-map-repeated-key-passes-file-not-env
- **Severity** — Medium
- **Confidence** — high
- **Files involved** —
  - `elixir_map.erl:96` (the offending call), for contrast `elixir_map.erl:132`, `elixir_map.erl:205`, `elixir_map.erl:228`
  - `elixir_errors.erl:285-289` (`function_error/4`), `elixir_errors.erl:264-269` (`file_error/4`)
- **Finding shape** — Type / contract mismatch across files (wrong argument type defeats a documented error-recovery contract)
- **Evidence** —
  - `elixir_map.erl:96`: in a *match* context, a repeated literal map key is reported with `function_error(Meta, ?key(E, file), ?MODULE, {repeated_key, Key})` — the **file binary** is passed as the second argument.
  - Every other `function_error` call in this module passes the **env map** `E`: `elixir_map.erl:132`, `:205`, `:228` (`function_error(Meta, E, ?MODULE, ...)`).
  - `elixir_errors.erl:286-289`: `function_error` only reaches its recoverable branch when the second argument matches `#{function := {_, _}}` (→ `module_error`, which taints the module and returns `ok`, allowing the fold to continue). A **binary** cannot match that map clause, so it falls to `function_error(Meta, Env, ...)` → `file_error(Meta, Binary, ...)` (`elixir_errors.erl:288`).
  - `elixir_errors.erl:264-269`: `file_error/4` calls `compile_error/1`, which is spec'd `-> no_return()` and **raises immediately**.
  - Caller context `elixir_map.erl:113-121`: `validate_kv` folds over all key/value pairs, calling `validate_not_repeated` and expecting it to *return* `Used` (`elixir_map.erl:99`) so later keys keep being validated. A raising `file_error` aborts the whole fold.
- **What the per-file pass missed** — A single-file reviewer of `elixir_map.erl` sees an argument inconsistency at worst; only by reading `elixir_errors.erl`'s `function_error`→`module_error`→`file_error` dispatch does it become clear that passing the file binary silently converts an intended "taint-and-continue" function error into a fatal, immediately-raising `file_error`, so only the first duplicate match key is reported instead of all of them.
- **Suggested fix** — Change `elixir_map.erl:96` to pass the env: `function_error(Meta, E, ?MODULE, {repeated_key, Key})` (matching lines 132/205/228).

#### `CSCAN-cluster-lexical-run-bootstrap-return-shape`

- **Finding ID** — CSCAN-cluster-lexical-run-bootstrap-return-shape
- **Severity** — Low
- **Confidence** — medium
- **Files involved** —
  - `elixir_lexical.erl:11-34` (`run/3`, the two asymmetric branches)
  - `elixir_module.erl:122-131` (caller that consumes `run/3`'s return value), `elixir_module.erl:126-127` (the `ExecutionCallback`/`AfterExecutionCallback` pair)
- **Finding shape** — State-machine / return-shape inconsistency across files (two branches of one function return different values; a caller depends on one shape)
- **Evidence** —
  - `elixir_lexical.erl:18-24` (non-bootstrap branch): after running the callbacks it evaluates `AfterExecutionCallback(LexEnv)` for side-effects and then returns `Res`, the **`ExecutionCallback` result**.
  - `elixir_lexical.erl:31-33` (bootstrap branch): `ExecutionCallback(E), AfterExecutionCallback(E)` — the block's value, and thus `run/3`'s return, is the **`AfterExecutionCallback` result**, not the `ExecutionCallback` result.
  - `elixir_module.erl:122-128`: when `lexical_tracker := nil`, `run/3` is the tail expression, so its value becomes `elixir_module:compile/6`'s return. The `ExecutionCallback` returns `compile(...)` = `{module, Module, Binary, Result}` (see `elixir_module.erl:247`), while the `AfterExecutionCallback` is `fun(_LexEnv) -> ok end` (`elixir_module.erl:127`).
  - Consequence: under `elixir_config:is_bootstrap() == true`, this path returns `ok` instead of the `{module, …}` tuple. The `nil`-tracker path is entered only when `Function /= nil` (`elixir_module.erl:117-120`), i.e. a `defmodule` nested inside a function/macro body during bootstrap — a narrow trigger, hence the confidence/severity.
  - The other caller, `elixir_compiler.erl:21-27`, discards `run/3`'s return entirely (it reads `elixir_module_binaries` instead), which is why the asymmetry has gone unnoticed.
- **What the per-file pass missed** — Reading `elixir_lexical.erl` alone, the two branches look interchangeable; the mismatch only surfaces when you see that `elixir_module.erl:122-131` uses `run/3`'s value as `compile/6`'s result while the other caller (`elixir_compiler.erl`) throws it away, so the bootstrap branch's differing return shape has an observable effect for exactly one of the two callers.
- **Suggested fix** — Make the bootstrap branch return the execution result symmetrically, e.g. `true -> Res = ExecutionCallback(E), AfterExecutionCallback(E), Res` in `elixir_lexical.erl:31-33`.
