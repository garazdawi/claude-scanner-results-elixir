# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_expand.erl
<!-- sha256: 1fcb8641377ffab1950db04e717abd15d11c0abd380f920ba4a72ee2d167011e -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_expand.erl
<!-- sha256: 1fcb8641377ffab1950db04e717abd15d11c0abd380f920ba4a72ee2d167011e -->

# Security review: `elixir_expand.erl`

_Re-reviewed on 2026-08-28 (third pass). Flagged hedges "did not benchmark" / "did not exhaustively" re-adjudicated by re-executing the amplification sweep with my own generator and by re-running the short-circuit axis. Both hedges resolved by execution below; no new assumption was upgraded on the strength of a read-only invariant._

## Summary

`elixir_expand.erl` is the Elixir macro expander — a defensively-complete tree walker whose catch-all clause turns every unrecognized AST node into a controlled `file_error`. It creates no atoms itself, performs no I/O, writes no global state, and deserializes nothing. Re-running (not re-reading) the two stress axes the original pass left un-benchmarked, I independently reproduced: recursion/expansion is linear in source-nesting depth, but the `__ENV__` → `escape_map` expansion is a **measured superlinear (≈O(source²)) compile-time amplification** — Finding 1 below, low confidence. It stays behind the compile/eval boundary the folder threat model documents as by-design code execution, hence low rather than a hard vulnerability.

The "did not exhaustively" hedge attached to two ruled-out entries: the malformed-AST crash (enumerated-and-ran; residual axis is combinatorially infinite and un-exhaustible) and the validation-predicate scope cross-product (asserted; a wrong answer there is a miscompile, not an authz bypass, so it is categorically not upgradeable to a security finding regardless of how many states I enumerate). Neither hedge could be honestly converted to `verified` by reading a neighbour, so per the re-review rules they stay where they are rather than being upgraded on a plausible invariant.

## Security

### Finding 1 (low) — `__ENV__` expansion is a superlinear compile-time resource amplification — **Upgraded from hedged assumption in first pass; benchmark re-executed in this pass**

- **Location:** `elixir_expand.erl:203-205` (bare `__ENV__`), `:206-213` (`__ENV__.field`), delegating to `escape_env_entries/3` (`:604-610`) and `escape_map/1` (`:612`).
- **Sink class:** Resource consumption / DoS — uncontrolled resource amplification (CWE-400 / CWE-1050).
- **Confidence:** Low.

**Mechanism.** Bare `__ENV__` (line 203-205) expands to `escape_map(escape_env_entries(...))`. `escape_env_entries/3` reads `Read` from `#elixir_ex{vars={Read,_}}` (line 604) and, via `escape_map/1` (line 612), emits a map literal `{'%{}', [], lists:sort(maps:to_list(Read))}` enumerating **every in-scope read variable**. `N` occurrences with `M` in-scope variables therefore emit `O(N·M)` AST from `O(N+M)`-sized source — quadratic amplification; the `lists:sort/1` adds `O(N·M log M)`, and the dominant remaining cost is eval-compiling the `N` generated `M`-entry map literals downstream. The amplifying AST is emitted here.

**Executed evidence — re-run independently in this pass** (Elixir 1.21.0-dev / OTP 29 [erts-17.0.2, jit]; my own generator: `fn -> a1=1 … aV=V; __ENV__ ×R end.()`, `Code.string_to_quoted!` timed separately from `Code.eval_quoted`):

| in-scope vars `v` | `__ENV__` refs `r` | src bytes | parse | expand+eval |
|---|---|---|---|---|
| 100 | 100 | 1696 | 0.6 ms | 171.2 ms |
| 100 | 200 | 2496 | 0.1 ms | 338.5 ms |
| 100 | 400 | 4096 | 0.1 ms | 677.6 ms | ← linear in `r` (2× → 2×) |
| 200 | 100 | 2796 | 0.1 ms | 460.2 ms |
| 400 | 100 | 4996 | 0.2 ms | 1370.2 ms |
| 800 | 100 | 9396 | 0.4 ms | 4207.6 ms | ← 8× `v` → ~24.6× time (superlinear) |
| 800 | 200 | 10196 | 0.3 ms | 8618.9 ms |

Parse cost is flat (<1 ms) throughout; the whole blowup is at expand+eval. A ~9.4 KB source reaches ~4.2 s; `v=800/r=200` reaches ~8.6 s. These are my own re-executed numbers and match the prior sweep within noise, so the "did not benchmark" hedge is resolved: the amplification is measured, not assumed.

**Fast-path note (re-executed this pass).** `__ENV__.<field>` (lines 206-213) resolves `Field` against the escaped env map. When `Field` is a real env key (`__ENV__.line`, `__ENV__.file`, …) line 211 returns the value directly with **no** amplification: I measured `v=800`, 100× `__ENV__.line` at **28.0 ms** versus **4207.6 ms** for bare `__ENV__` — a ~150× gap confirming the short-circuit. When `Field` is *unknown*, line 212 emits the full `escape_map(Env)` AST but the eval then raises `KeyError` (Macro.Env has no such key) — so the AST blowup is emitted at expand time but the form is not a clean runnable vector. **Bare `__ENV__` is the reliable amplifier.**

**Why low / dominated.** Expansion runs only during compilation/eval, which the folder threat model documents as by-design code execution: any actor who can drive `elixir_expand` on this source already holds compile-time + runtime code execution, so the amplification is strictly dominated by capability they already have. `Code.string_to_quoted/2` (parse only) does **not** expand `__ENV__` — confirmed again by the flat parse column — so a parse-only untrusted-input path cannot trigger it. Robustness/amplification wart, not a clean-boundary vulnerability. Hence Low.

## Risks / caveats

- **Cross-file dependencies relied on** (enumerated in *Did I verify this?*): atom-table safety depends on `elixir_aliases:concat/1` being fed only source-derived alias parts; by-design macro dispatch lives in `elixir_dispatch`; raw-byte sanitization is upstream in `elixir_tokenizer`. These remain `asserted` — I did not clear their unexamined axes, and per the re-review rules reading those modules would not clear them.
- The one uncaught Erlang error I could produce (`ArgumentError` via `lists:keyfind/3` in `elixir_utils:var_context/2`, reached from `elixir_expand.erl:418/450`) requires a non-list `Meta`, which violates the Macro-AST contract and cannot be produced by the tokenizer/parser (see Ruled out); it crashes only the caller process.

### Ruled out

- **Code execution (macro dispatch)** — `elixir_dispatch:dispatch_import` / `dispatch_require` (lines 528, 542) invoke macros at compile time; documented by-design compiler behavior, not a sink introduced here.\
  Axes examined: expansion re-entry of macro output, local vs remote dispatch | Not examined: dispatch-internal allow-listing, macro sandboxing\
  Basis: asserted — Depends on: elixir_dispatch.erl. (Reading it would not clear the un-examined axes, so it stays asserted rather than being falsely promoted to verified.)

- **Atom creation / atom-table exhaustion** — `elixir_aliases:concat/1` (lines 621, 624, 1041, 1088) uses `binary_to_atom/2` on `__aliases__` parts that are themselves source atoms. Each distinct alias expression yields one atom → bounded O(source), no amplification.\
  Axes examined: multi-alias `Foo.{A,B,...}` fan-out, repeated-alias, non-existing-module atom creation | Not examined: interaction with `binary_to_atom` 255-char / atom-table limit under a crafted multi-MB source\
  Basis: asserted — Depends on: elixir_aliases.erl. The residual exhaustion axis is a general compiler property owned by `elixir_aliases`; I did not execute a multi-MB source to try it, so it stays on `Not examined` and the entry stays asserted.

- **Deserialization (`binary_to_term` etc.)** — none present.\
  Axes examined: full-file grep for `binary_to_term`, `file:consult`, `dets`, `disk_log` | Not examined: none\
  Basis: verified.

- **Shared mutable state / process dictionary** — only reads: `maps:get`/`map_get` on the env map (lines 211, 269) and `elixir_config:get(on_undefined_variable)` (line 496, operator-controlled config). No `put`/`erase`/`ets:`/global write anywhere.\
  Axes examined: full-file grep for `put(`, `erase(`, `ets:`, `!`, `send` | Not examined: none\
  Basis: verified.

- **Command / file / network / NIF / crypto sinks** — none. Only filesystem-adjacent call is `filename:dirname(?key(E,file))` for `__DIR__` (line 194), a pure read-only string op on the compiler-supplied file path.\
  Axes examined: full-file grep for `os:`, `open_port`, `file:`, `filelib:`, `spawn`, crypto, `load_nif` | Not examined: none\
  Basis: verified.

- **Validation predicates (wrong-answer)** — `assert_no_*` scope gates (194-215, 1136-1165), `should_warn`, `is_loaded_and_not_exported` (1008), `validate_for_options` (881-899), `validate_opts` (1025-1031). A wrong answer yields a miscompile or confusing diagnostic, not a privilege/authz bypass; none is a security boundary.\
  Axes examined: match/guard-scope confusion, opts non-list, uniq/reduce/into conflict | Not examined: exhaustive scope-state cross-product\
  Basis: asserted — Depends on: elixir_clauses.erl / elixir_env.erl. This is the "did not exhaustively" hedge: even fully enumerating the scope-state cross-product could at most reveal a miscompile/diagnostic error, which is categorically not a security finding, so the entry cannot be upgraded and reading the neighbours cannot move it to verified. It stays asserted — the safe direction.

- **Uncaught crash on malformed AST (robustness/DoS)** — hand-crafted `{:foo, :bar, :baz}` (non-list `Meta`) reaches `elixir_utils:var_context/2` → `lists:keyfind(counter, 1, :bar)` → `ArgumentError`. Requires an AST violating the Macro contract (Meta must be a keyword list); the parser never emits non-list Meta, so unreachable from source and out of scope (contract violation); crashes only the caller process.\
  Axes examined (ran): non-list Meta, non-list Args (→ catch-all `invalid_call`), `make_ref`/pid/fun literals (→ `invalid_quoted_expr`), empty `<<>>` generator, empty `__block__`, deep nesting, pin/alias/bind_quoted misuse | Not examined: every hand-crafted 3-tuple permutation (combinatorially unbounded — cannot be exhausted, so it remains an open axis by construction, not by omission)\
  Basis: verified for the enumerated axes (ran). The residual "every permutation" axis is un-exhaustible; a worst case there is still only a caller-process crash, which the folder threat model scopes as out-of-bounds (contract violation), so the residual does not change the ruling.

- **Resource consumption — recursion / expansion depth** — `expand/3` recursion depth = source nesting depth (shared by parser; caller-process crash only). Executed: `Code.string_to_quoted! |> Code.eval_quoted` on `((…(1)…))` at nesting 100/500/1000/2000/4000 → 0.5 / 0.1 / 0.2 / 0.5 / 1.1 ms — flat/linear, no superlinear blowup, no stack failure at 4000-deep. The `escape_map`-per-`__ENV__` cost that formerly shared this entry is promoted to Finding 1.\
  Axes examined: deep parenthesization/nesting depth (ran, 100→4000, linear) | Not examined: none\
  Basis: verified (ran; nesting axis is linear and caller-scoped).

- **Counter rollover** — `version`/`Counter` fields (lines 343, 431, 440, 819, 877, etc.) are Erlang arbitrary-precision integers; no wrap.\
  Axes examined: increment sites for cond/case/receive/try/for and per-var version | Not examined: none\
  Basis: verified.

- **Template / human-in-the-loop (`format_error`)** — error formatters use `io_lib:format` with `~ts`/`~s` on `'Elixir.Macro':to_string/1` / `inspect/2` output (bounded binaries); no value reaches a downstream interpreter and no `~`-directive is built from external data.\
  Axes examined: format-string injection, unbounded expansion of formatted term | Not examined: none\
  Basis: verified.

### Did I verify this?

- **Finding 1 (`escape_map`/`__ENV__` amplification)** — `verified as present`. **Re-executed independently in this pass** with my own generator: the parameter sweep tabulated above (parse isolated, flat <1 ms; expand+eval linear in `r`, superlinear in `v` — 8× `v` → ~24.6× time; ~4.2 s at ~9.4 KB, ~8.6 s at v=800/r=200). Also re-ran the fast-path axis: `__ENV__.line` (real key) = 28.0 ms vs bare `__ENV__` = 4207.6 ms at v=800, and `__ENV__.<unknown>` raises `KeyError` at eval. Read lines 203-213 and 604-612 to confirm `escape_map` enumerates the full `Read` var map per occurrence and that the real-key branch (line 211) short-circuits. "Did not benchmark" hedge fully resolved. `Not examined: none` on the sweep and fast-path axes.
- **Resource consumption — nesting axis** — `verified`. Ran depth 100–4000; 0.1–1.1 ms, no superlinearity, no stack overflow. `Not examined: none`.
- **Deserialization / shared state / command-file-network-NIF-crypto / counter rollover / format_error** — `verified`. Full-file grep of the exact sink patterns returned only the three benign reads at lines 211/269/496; read `format_error/1` (1169-1365). `Not examined: none`.
- **Uncaught crash on malformed AST** — `verified` for the enumerated axes (ran): `for <<>>, do: 1` → `CompileError`; `make_ref()` AST → "invalid quoted expression"; `{:foo,:bar,:baz}` → `ArgumentError` "3rd argument: not a list"; `^x`, `alias 1.{Foo}`, bad `bind_quoted`, 50-deep parens, `case (;) do _ -> 1 end`, `__ENV__.unknown` all produced clean controlled errors. Residual "every 3-tuple permutation" is un-exhaustible; worst case is a caller-process crash (out of scope).
- **Validation predicates** — remains `asserted`. This is the "did not exhaustively" hedge. Read the gate clauses; did not trace the full downstream scope-state cross-product in `elixir_clauses`/`elixir_env`, and — critically — a wrong answer there is a correctness/diagnostic issue, not an authz boundary, so it cannot become a security finding no matter how the cross-product resolves. Reading the neighbours cannot clear this axis per the re-review rules. `Depends on: elixir_clauses.erl / elixir_env.erl`.
- **Code execution (macro dispatch)** — remains `asserted`. Compile-time macro invocation lives in `elixir_dispatch`; it is the intended, documented trust boundary (eval runs code). Reading it would not clear the un-examined dispatch-internal axes. `Depends on: elixir_dispatch.erl`.
- **Atom creation** — remains `asserted`. Confirmed `elixir_aliases:concat/1` uses `binary_to_atom/2` by reading `elixir_aliases.erl`; sink is real but bounded by distinct source aliases (O(source), no amplification). The multi-MB-exhaustion axis was not executed, so it stays on `Not examined`. `Depends on: elixir_aliases.erl`.

### Dropped in re-review

- Nothing dropped. Finding 1 is preserved and its benchmark re-executed independently (numbers match the prior sweep within noise). All ruled-out entries are preserved with their basis unchanged; the two "did not exhaustively" entries stay at `asserted`/`verified-for-enumerated-axes` respectively rather than being upgraded, because in both cases the un-examined axis either (a) cannot be cleared by reading a neighbour, or (b) is categorically incapable of producing a security finding. Per the re-review rules, no dismissal was moved from `asserted`→`verified` or from dismissal→finding without executed evidence.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **"Resource consumption" ruled-out entry (former combined form)** — not dropped, but **split**. Its recursion/nesting sub-axis is retained above as `verified` (ran the depth sweep). Its `escape_map`/`__ENV__` sub-axis is **removed from Ruled out and promoted to Finding 1** because executed benchmarking showed a genuine superlinear (≈O(source²)) amplification, contradicting the first pass's "semantically required… not an amplification" dismissal. Nothing was silently dropped; the assessment moved from dismissal → low-confidence finding, the safe direction under the re-review rules.
