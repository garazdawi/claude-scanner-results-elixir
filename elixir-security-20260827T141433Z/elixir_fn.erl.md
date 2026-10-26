# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_fn.erl
<!-- sha256: b505954c645390e3d4cce0e187e3d58e7785f7a15e8c31901e3b040a5941a2f4 -->

_Analysed on 2026-08-27T17:06:00Z_

Definitive. The isolation is conclusive:

- **Test A** (`&(&1 + &1 + … + &1)`, N terms, all position 1 → orddict stays size 1, yields a valid arity-1 fn that is *fully* compiled): **linear** — 3200 terms = 2.9 ms.
- **Test B** (`&(&1 + &2 + … + &N)`, N terms, distinct positions → orddict grows to N, fn errors early on arity): **quadratic-plus** — 400→61 ms, 800→391 ms, 1600→4.4 s, 3200→**32 s**.

Same tree size; the only variable is whether the `&K` positions are distinct, i.e. whether the `orddict` in `escape/3` grows. That pins the blowup precisely on lines 153/159, and test A proves the downstream fn machinery is linear. `next_counter` handles `nil` and never atomizes (read `elixir_module.erl:64-71`).

Here is my report.

---

#### `CSCAN-elixir_fn-capture-orddict-quadratic`

- **Finding ID** — `CSCAN-elixir_fn-capture-orddict-quadratic`
- **Severity** — Medium
- **Confidence** — high
- **Reachability** — reachable
- **CVSS** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:L` → 5.3 (Medium). (A:L reflects that a single request self-terminates after burning input²-proportional CPU; sustained/large or repeated inputs approach A:H. AV:N reflects the realistic deployment: a service that compiles/evaluates user-submitted Elixir source. On a purely local `elixirc` invocation this is AV:L.)
- **Location** — `elixir_fn.erl:149-176` (`escape/3`), specifically `orddict:find/2` at line 153 and `orddict:store/3` at line 159; the missing upper bound in the `Pos > 0` guard at line 149; `validate/4` at lines 142-147 also lacks an arity cap. Contrast the correct `0 =< A =< 255` cap in `args_from_arity/3` at line 178.
- **Class** — Resource consumption (quadratic algorithmic complexity / CPU + allocation DoS).

- **Attack scenario** — An `unauthenticated network peer` submits a small Elixir source string to any `library consumer` that compiles or evaluates it — an online Elixir playground, a chatbot that runs user snippets, a LiveBook/notebook backend, an EEx-templating service, a language server that expands a file on save, or a build server compiling a third-party dependency (supply-chain). The source contains a single capture expression with N *distinct* positional arguments, e.g. `&(&1 + &2 + … + &N)` (equivalently `&foo(&1, &2, …, &N)` or `&[&1, …, &N]`). During macro expansion, `elixir_fn:escape/3` walks the expression and threads an `orddict` accumulator through it, calling `orddict:find/store` once per `&K`; because the keys are distinct and increasing, each op is O(current size), so the pass is O(N²) in time and allocates O(N²) list cells. A ~25 KB source (N≈3200) consumes ~32 s of single-core CPU plus heavy GC before the eventual (irrelevant) arity error; the cost scales quadratically, so ~50 KB ≈ 2 min, ~100 KB ≈ 8 min. **New capability:** a submitter whose input previously cost work *linear* in its size can now force the victim's compile/eval process into minutes of 100%-CPU, quadratic in a tiny input — a denial-of-service lever they did not have from linear-cost submissions.

- **Trace** — `elixir_expand` dispatches the `&` operator to `elixir_fn:capture/4`. For `&(&1 + &2 + … + &N)`, capture matches the `{Atom,_,Args}` import clause (line 67) → `capture_import` (non_sequential, line 82) → `capture_expr/5` (line 118) → `capture_expr/6` (line 124) → `escape(Expr, E, [])` (line 125). `escape/3` recurses over the `+` tree (lines 165-174); each `&K` leaf hits the `is_integer(Pos), Pos > 0` clause (line 149), does `orddict:find(Pos, Dict)` (line 153, miss → O(size)) then `orddict:store(Pos, Var, Dict)` (line 159, insert at tail → O(size)). With N distinct increasing positions the accumulator grows to N entries → Σ O(K) = O(N²). `validate/4` (line 134/142) imposes no arity cap, so escape runs to completion for any N before the arity->255 rejection happens downstream in the re-expanded `fn`.

- **Boundary** — Untrusted source text crossing into the compiler front-end via a documented public entry point (`Code.eval_string/2,3`, `Code.compile_string/2`, `Code.eval_quoted/2,3`, `Code.string_to_quoted/2` + expansion, `elixir:'string_to_quoted!'/5`, or plain compilation of an attacker-supplied `.ex`/`.exs`). The folder threat model names source text as "the primary untrusted input" and lists "Quadratic blowup … on large input" and "DoS of the compiling process (… unbounded CPU/memory …)" as in scope.

**Impact** —
- Confidentiality: none.
- Integrity: none.
- Availability: a single small input (tens of KB) ties up one BEAM scheduler for tens of seconds to minutes at 100% CPU with heavy GC churn (O(N²) transient allocation). On a service that compiles/evals untrusted source without a CPU/wall-clock sandbox (build servers, language servers, notebook/eval backends), a few such requests saturate all workers and deny service. Blast radius is the compiling process (a parse/expand crash kills only the caller per the threat model), but the CPU/latency cost is real and quadratic. Self-limiting per request (the >255-arity fn eventually errors), but the CPU is already spent and the attacker can repeat.

- **Validation** — Ran against the installed OTP 29 / Elixir 1.21.0-dev whose `elixir_fn.beam` is compiled from the byte-identical source (`diff` reported IDENTICAL).
- `ran /opt/elixir/bin/elixir qtest3.exs` — baseline `1 + 2 + … + N` (same-shape tree, no capture): `N=100 0.5ms … N=1600 0.9ms` (flat/linear); capture `&(&1 + … + &N)`: `N=100 1.1ms, N=200 10.1ms, N=400 62.6ms, N=800 393.5ms, N=1600 4433.7ms` (≈4–11× per doubling → super-quadratic).
- `ran /opt/elixir/bin/elixir qtest4.exs` — isolation holding tree size constant: all-`&1` (orddict size 1) `terms=3200 → 2.9ms ok=true`; distinct `&1..&N` (orddict grows) `terms=400 61.7ms, 800 391ms, 1600 4437ms, 3200 32360ms`. The only difference is orddict growth → the O(N²) is `escape/3`'s orddict, not the downstream fn machinery (which test A exercises fully and linearly).
- `read elixir_fn.erl:149-181` — `escape` `Pos > 0` clause has no upper bound and stores into `orddict`; `args_from_arity` (line 178) does cap at 255, so the positional `&N` path is the unguarded twin.

- **Suggested fix** — Two options, ideally both:
1. Minimal/targeted: cap positional captures at the real BEAM arity limit in the `escape/3` guard, mirroring `args_from_arity/3`. Change line 149 to `... when is_integer(Pos), Pos > 0, Pos =< 255 ->` and add a clause rejecting `Pos > 255` with an `invalid_arity_for_capture`-style error. This bounds the orddict to ≤255 entries, so find/store stay O(255) and the whole pass is linear; it also reports the arity violation eagerly instead of after O(N²) work.
2. Algorithmic: replace the `orddict` accumulator in `escape/3` with a map (`#{}`; `maps:find/2`, `Dict#{Pos => Var}`) for O(1) amortized find/store, and in `capture_expr/6` sort the positions once before `validate` (`lists:keysort(1, maps:to_list(EDict))`). This makes the pass O(N log N) regardless of arity.

**Evidence** —
`read /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_fn.erl:142-181` — `escape/3` orddict find/store, `validate/4` with no cap, `args_from_arity/3` capping at 255.
`ran diff -u /opt/elixir/.../elixir_fn.erl <target>` — "IDENTICAL — installed beam is a faithful test oracle".
`ran /opt/elixir/bin/elixir qtest3.exs` — capture path `N=1600 → 4433.7ms` vs baseline `N=1600 → 0.9ms`.
`ran /opt/elixir/bin/elixir qtest4.exs` — distinct-position `terms=3200 → 32360.5ms` vs all-`&1` `terms=3200 → 2.9ms`.
`read /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_module.erl:64-71` — `next_counter(nil) -> erlang:unique_integer()`, no atom creation.

---

### Ruled out

- **`escape/3` recursion depth (lines 165-174)** — Unbounded recursion over AST depth (linear in nesting).
Reason — Depth = source nesting depth, which the tokenizer/parser and `elixir_expand` recurse over first; Erlang grows the process stack dynamically and, at the limit, raises a catchable `system_limit` in the caller's own process, not memory corruption. Test B recursed 3200-deep with no stack fault; the orddict cost dominates and manifests long before any stack limit. General compiler property, subsumed by the quadratic finding.
Axes examined: left-nested tree, distinct/duplicate positions, depth 3200 | Not examined: extreme depth (≥10⁶) stack-limit crash, right-nested vs left-nested balance
Basis — asserted
Depends on: elixir_tokenizer.erl / elixir_parser.yrl — the parser recurses to at least the same depth to build the AST before `escape` sees it, so `escape` is not the first blow-up point; and the enclosing `elixir_expand:expand` recurses over the same tree.

- **`fn_arity/1` `length(Args) - 1` (line 41)** — Arity computation for `fn` clauses.
Reason — On a `[{'when', _, []}]` shape it would yield -1, but the result is only compared via `lists:usort` for "all clauses same arity"; a bogus number causes at most a spurious `clauses_with_different_arities` error, no crash or resource issue. Post-expansion `when` always carries ≥1 element (the guard), so -1 is not even producible from parsed source.
Axes examined: empty when-args, single guard, mixed arities | Not examined: none
Basis — verified

- **`capture/4` clause dispatch on malformed AST (lines 12-14, 28, 46-80)** — `lists:any/length/mapfoldl` over `Left`/`Args`/`Clauses` assume list shape; a non-list would `function_clause`/`badarg`.
Reason — From the documented source-parsing path the grammar guarantees `fn` clauses are `{'->', _, [ListArgs, Body]}` and capture args are lists; producing a non-list requires macro-generated or hand-built AST fed to `Code.eval_quoted`, i.e. violating the AST contract — out of scope (contract/`-spec` violation, and eval of untrusted AST is a documented caller bug).
Axes examined: non-`->` clause, non-list args, 2-tuple clause | Not examined: none (out-of-scope by contract)
Basis — verified

- **`args_from_arity/3` arity guard (lines 178-181)** — Defensive `0 =< A =< 255` cap.
Reason — Reachable and correct: `&Mod.fun/256` and `&Mod.fun/-1` both fall through to the `file_error` clause; the bound matches the BEAM max arity. This is a working guard, not dead code — and its absence in the parallel `escape` positional path is captured by the finding above.
Axes examined: A>255, A<0, A=0, A=255, bignum A | Not examined: none
Basis — verified (`ran` — `&Foo.bar/256` triggers `invalid_arity_for_capture`; confirmed by reading lines 178-181)

- **`elixir_module:next_counter/1` (line 157)** — Shared-state counter / potential atom creation.
Reason — Returns an integer (`ets:update_counter`) or `erlang:unique_integer()` for `nil`/error; creates no atoms and cannot exhaust the atom table. The variable atom `'_&'` (line 158) is a fixed literal. The ETS counter is monotonic (Erlang bignum, no overflow) and per-module; bumping it N times is not a poisoning vector.
Axes examined: module=nil, module=atom, ETS-missing (catch), counter growth | Not examined: none
Basis — verified (`read elixir_module.erl:64-71`)

- **`format_error/1` → `'Elixir.Macro':to_string/1` with `~ts`/`~s`/`~B` (lines 199-232)** — Attacker-controlled AST stringified into compiler diagnostics (template / human-in-the-loop).
Reason — Format strings are compile-time constants and every interpolated value is passed as a `~`-directive argument, so there is no format-string injection. `~B` targets (`invalid_arity_for_capture`, `capture_arg_without_predecessor`, `capture_arg_outside_of_capture`) are all guaranteed integers by the `is_integer` guards that reach them, so no `badarg`. `Macro.to_string` emits canonical Elixir syntax rather than echoing raw source bytes, and the tokenizer already rejected bidi/`?break` controls before an AST exists, so a Trojan-Source-style diagnostic smuggle is not introduced here. At most a defense-in-depth concern (a human reading a diagnostic that reflects hostile-looking identifiers).
Axes examined: `~B` non-integer arg, atom vs chardata for `~ts`/`~s`, bignum position, bidi/escape in stringified AST | Not examined: exhaustive `Macro.to_string` output-shaping (cross-file)
Basis — asserted
Depends on: elixir_tokenizer.erl — `?bidi`/`?break` rejection removes directional/line-break controls from source before any AST reaches `format_error`; `Elixir.Macro.to_string/1` returns well-formed chardata (never a bare atom that would `badarg` a `~ts`).

- **`elixir_dispatch:import_function/require_function` (lines 84, 104)** — Function resolution by name/arity.
Reason — The name (`Atom`/`Right`) is already an atom produced by the parser and the arity is `length(Args)` (bounded by source size); no dynamic `list_to_atom`/`binary_to_atom` occurs in this file, and dispatch is a lookup, not a sink. Cross-file behavior, not a sink here.
Axes examined: name origin (parsed atom), arity value | Not examined: internal behavior of dispatch (out-of-file)
Basis — asserted
Depends on: elixir_dispatch.erl — `import_function`/`require_function` do not atomize untrusted text and return `false`/a resolved tuple without unbounded allocation.

- **`elixir_expand:expand/3` on the module part (line 90)** — Re-entrant expansion of the `Left` of `&Left.fun/arity`.
Reason — Delegates back to the general expander (in-folder callee `elixir_expand`); any cost/recursion there is the expander's own general property, not a sink introduced by this file.
Axes examined: variable/atom/complex module expr | Not examined: expander internals (out-of-file)
Basis — asserted
Depends on: elixir_expand.erl — `expand/3` terminates and does not itself introduce super-linear cost on the module sub-expression.

### Did I verify this?

- **`escape/3` recursion depth** — kept `asserted`. `Not examined` still lists the extreme-depth (≥10⁶) stack-limit case, which I did not drive, so this is not `verified`. The `Depends on:` line names the parser/expander as the prior-recursion mechanism; the cross-file pass should confirm the parser recurses to at least this depth first (making `escape` non-first-to-blow), otherwise a pure deep-nesting stack-exhaustion vector is a separate Low.
- **`fn_arity/1`** — `verified`; `Not examined: none`; the impact (a number into `usort`) is fully contained in-file.
- **`capture/4` malformed-AST dispatch** — `verified` as out-of-scope: reaching it needs an AST that violates the grammar/`-spec`, only producible by macro-generation or hand-built AST + `eval_quoted`, which the scope rules exclude.
- **`args_from_arity/3` guard** — `verified`; `Not examined: none`; ran the boundary (`/256`, `/-1`) and read the clause.
- **`next_counter/1`** — `verified`; read `elixir_module.erl:64-71`; no atom creation, `nil` handled, `Not examined: none`.
- **`format_error/1` diagnostics** — kept `asserted`. I verified the `~B` integer-only reachability and the constant-format-string structure in-file, but the claim that no bidi/confusable content can reach a human-read diagnostic depends on `elixir_tokenizer`'s `?bidi`/`?break` gate (out-of-file); the `Depends on:` line makes that contract machine-extractable for the cross-file pass.
- **`import_function`/`require_function`** and **`elixir_expand:expand/3`** — kept `asserted`; both dismissals rest on the named callee modules not atomizing untrusted text and terminating without super-linear cost. The `Depends on:` lines name exactly where the cross-file pass must confirm those contracts.

### Risks / caveats
- The quadratic finding is a CPU/allocation DoS only — no confidentiality, integrity, memory-safety, or code-execution consequence, and the damage is confined to the compiling process (the folder threat model notes a parse/expand crash kills only the caller). Its real-world weight depends on deployment: services that compile/evaluate untrusted Elixir source *without* a CPU/wall-clock sandbox (build servers pulling dependencies, language servers, notebook/eval backends, EEx template compilers) are exposed; a local developer compiling their own trusted code is not a victim.
- Timing was measured on the byte-identical installed build (OTP 29 / Elixir 1.21.0-dev), verified via `diff`. Absolute numbers depend on hardware, but the *scaling* (linear baseline vs. quadratic-plus capture, isolated to orddict growth) is machine-independent and reproduced across two independent test scripts.
- Several ruled-out dismissals (`format_error` bidi-safety, dispatch atomization, `elixir_expand` termination, deep-recursion-not-first) carry explicit `Depends on:` contracts on neighbouring modules (`elixir_tokenizer`, `elixir_dispatch`, `elixir_expand`); if any of those contracts does not hold, the corresponding sink should be revisited.

