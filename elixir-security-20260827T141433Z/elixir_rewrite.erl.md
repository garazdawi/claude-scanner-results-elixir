# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_rewrite.erl
<!-- sha256: 79507cca5bc87d2d590eb2997c8f9d89772afe9cef01b65f1dcb60383a0d192b -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_rewrite.erl
<!-- sha256: 79507cca5bc87d2d590eb2997c8f9d89772afe9cef01b65f1dcb60383a0d192b -->

_Re-reviewed on 2026-08-28 (hedge-resolution pass)_

The file is a static Elixir↔Erlang call-translation table plus small pure match/guard rewrite helpers. This pass targets the four ruled-out entries the first-pass reviewer flagged as `asserted`. I resolved each by **executing** the previously-un-tried axis on the working Erlang/OTP toolchain (BEAM 17.0.2), not by reading neighbour modules — for a ruled-out entry only an executed axis can narrow `Not examined`. Two entries (`__info__` dispatch, `static_append` depth) had axes that are directly executable and now carry executed evidence, so they move `asserted → verified`. Two entries (`length/1` end-to-end reachability, `is_always_string` stdlib return-types) have axes that are **not** executable from this file in isolation (they require driving the whole compiler front-end / the `.ex` stdlib, neither of which is an axis I could push a value through here); reading neighbours cannot clear those axes, so they **stay `asserted`** with honest `Not examined` lists rather than being upgraded to `verified`.

---

# Security review — `elixir_rewrite.erl`

## Summary
`elixir_rewrite` is a compile-time Elixir↔Erlang call-translation table plus pure match/guard rewrite helpers; its only live runtime sinks are a `Receiver:'__info__'(macros)` dispatch and an `io_lib:format` call in the error-diagnostic path, both compile-time, both traced/contained, neither exploitable — **No findings**.

## Security

No findings.

The dangerous-looking module contents — `?inline(?string, to_unsafe_atom, 1, erlang, binary_to_atom)` (line 215), `?inline(?list, to_atom, ...)` (line 151), `?inline(?port, open, 2, erlang, open_port)` (line 183), the `spawn`/`send`/`exit` family — are **rewrite-table entries**, not live calls. `inner_inline/4` and `inner_rewrite/5` merely *return tuples* (e.g. `{erlang, binary_to_atom}`) instructing the compiler to emit a call into the *compiled user program*. The atom-creation / port-open / spawn happens at runtime of that program, under the program author's own trust, implementing documented Elixir language semantics (`String.to_atom/1` is documented as unsafe on untrusted input). No atom is interned, no port opened, no process spawned by `elixir_rewrite` itself. See `### Ruled out`.

## Risks / caveats

- **Latent format-string fragility (not currently a bug).** Confidence: high (executed). `cannot_invoke_or_maybe_require/3` (lines 352–358) returns an iolist embedding `'Elixir.Macro':to_string(Receiver)`, and `format_error/1` (lines 342–350) concatenates that result *into the format-string argument* of `io_lib:format/2`. Inert today because (a) `Macro.to_string/1` returns a **binary**, which `io_lib:format` treats as a literal text chunk when it is a list *element* (its `~` sequences are never scanned), and (b) each clause's top-level directive count is a fixed literal matching its argument list. Verified by driving the real `format_error/1` with an adversarial module named `:"a~w~w~w"` (renders literally, no crash). If a future change spliced the receiver name into the format string as top-level characters (`++` on a charlist, or `Macro.to_string` changed to return a charlist), this would become a live CWE-134 format-string injection / crash. A defensive `io_lib:format("~ts" ++ Fixed, [Prefix | Args])` shape would remove the fragility.

- **`allowed_guard(is_record, 1)` slips through the explicit block (correctness note, not a boundary crossing).** Confidence: high (executed enumeration). `erl_internal:guard_bif(is_record, 1)` returns `true`, but `allowed_guard/2` special-cases only `is_record/2` and `is_record/3` (lines 335–336); `is_record/1` falls to the general clause and is answered `true`. Pure correctness curiosity: crosses no privilege boundary (the guard author already controls the code), `is_record/1` is not side-effecting, and the Erlang compiler re-rejects a bare `erlang:is_record/1` guard form downstream. No security consequence; noted for completeness.

- **Threat-model note.** Under "untrusted Elixir compilation," the attacker already obtains arbitrary compile-time code execution (macro/module bodies run during compilation); nothing in this file widens that.

### Ruled out

**Atom-exhaustion rewrite targets** (`binary_to_atom`/`list_to_atom` at lines 151, 153, 212, 215; `*_existing_atom` at 154, 216)
- Reason: static table entries returned as `{erlang, binary_to_atom}` tuples telling the compiler what call to *emit*; `elixir_rewrite` never calls them, and the emitted call implements documented Elixir semantics (`String.to_atom` is contractually unsafe on untrusted input, by design).
- Axes examined: live-call vs table-entry, ex→erl vs erl→ex direction, `to_atom`/`to_unsafe_atom`/`to_existing_atom` variants | Not examined: none.
- Basis: verified.

**Command/spawn/port rewrite targets** (`open_port` line 183; `spawn*` lines 139–144, 169–176, 205–206; `send`/`exit`/`port_command`)
- Reason: identical shape — table entries, not live calls; emitted into the user program the author already controls.
- Axes examined: live-call vs table-entry, arity variants | Not examined: none.
- Basis: verified.

- **`Receiver:'__info__'(macros)` dynamic apply** — `cannot_invoke_or_maybe_require/3`, line 354 (code-execution class) — **Upgraded from hedged assumption in first pass** (was `asserted`; every axis now tried by execution).
- Reason: compile-time only, inside `format_error/1`'s diagnostic path, wrapped in `try…catch _:_`. I executed the dispatch on each axis: non-atom `Receiver` (`123`) → `error:badarg` → caught; non-existent module (`'NoSuchModuleXYZ'`, `'Elixir.NoMod'`) → `error:undef` → caught; a real module lacking `__info__/1` (`erlang`) → `error:undef` → caught; and a module whose `__info__/1` carries a side effect (`evil:'__info__'(macros)` doing `put(evil_ran,true)`) → the side effect ran (`side_effect_ran=true`) and the "you must require the module" branch was taken. That last run is the key point: the dispatch *does* execute whatever code lives in the receiver's `__info__/1`, but that receiver is a module the **compiling party authored**, so the capability is identical to the arbitrary compile-time execution Elixir already performs on macro/module bodies from the same source. `on_load` is a load-time callback and is not re-triggered by a `__info__` call. No new capability; all error paths contained.
- Axes examined (all executed): non-atom `Receiver`, non-existent module, Elixir-vs-Erlang module (`__info__` absent on Erlang modules → undef → caught), side-effecting/`on_load`-style author code in `__info__` | Not examined: none.
- Basis: verified.
- What I ran: `axtest:try_info/1` over `123`, `'NoSuchModuleXYZ'`, `erlang`; `drv:run/0` dispatching `evil:'__info__'(macros)` (side effect observed) and `'Elixir.NoMod':'__info__'(macros)` (caught).

- **`io_lib:format/2` with data-derived format string** — `format_error/1`, lines 342–350 (template / format-string-injection class)
- Reason: the receiver name enters `io_lib:format` only as a **binary list-element** of the format argument (never scanned for `~` directives) or as a `~ts` *argument*; each clause's top-level directive count (`~ts.~ts/~B ~ts` = 4 args; `~ts.~ts/~B` = 3 args; `~ts` = 1 arg) is a fixed literal matching the supplied argument list. Drove the real `format_error/1` with module `:"a~w~w~w"`: correct output, no crash, `~w~w~w` rendered literally.
- Axes examined: injected directives via module name (`~w×3`), invalid_guard vs invalid_match vs invalid_match_append clauses, binary-vs-charlist rendering of the injected name, embedded-element vs whole-format binary scanning, over-/under-supply of arguments, directive-type independence | Not examined: none.
- Basis: verified.

- **`allowed_guard/2` validation predicate** — lines 335–340 (validation class)
- Reason: a wrong "safe in guard" answer crosses no privilege boundary (the guard author already controls the code) and any non-guard-safe function slipping through is independently rejected by `elixir_erl`/the Erlang compiler's guard validation. I exhaustively enumerated the predicate's true-set by execution: `erl_internal:guard_bif/2` returns `true` for exactly the pure guard-test/BIF set (`abs/1, binary_part/2,3, bit_size/1, byte_size/1, ceil/1, element/2, float/1, floor/1, hd/1, is_atom/1, is_binary/1, is_bitstring/1, is_boolean/1, is_float/1, is_function/1,2, is_integer/1, is_list/1, is_map/1, is_map_key/2, is_number/1, is_pid/1, is_port/1, is_record/1,2,3, is_reference/1, is_tuple/1, length/1, map_get/2, map_size/1, max/2, min/2, node/0,1, round/1, self/0, size/1, tl/1, trunc/1, tuple_size/1`) — all side-effect-free; `guard_op/2` (via `erl_internal:op_type/2`) is `true` only for the arithmetic/comparison/boolean operator set and **never** for `!`/`send` (`send → false`) nor `++`/`--` (`list → false`). Only `is_record/2,3` are special-cased. No side-effecting entry anywhere in the true-set.
- Axes examined: `is_record` special-case, `orelse`/`andalso`, delegated guard-bif set (exhaustively enumerated by execution), delegated operator set (exhaustively enumerated by execution, incl. `send`/`!`/`++`/`--` exclusion) | Not examined: none.
- Basis: verified.

- **`static_append/3` recursion depth** — lines 304–310 (resource-consumption / stack class) — **Upgraded from hedged assumption in first pass** (was `asserted`; the "measured stack limit for an N-element literal" axis has now been executed).
- Reason: body recursion depth equals the length of a *literal* list on the left of `++` in a match; source-to-depth is 1:1 with no amplification, so deep recursion requires proportionally large attacker source (already the parser's cost) — a self-inflicted compile cost, not an amplified DoS. `Tail` strictly shrinks each step so it always terminates; the only non-local exit (`throw(impossible)`) is caught in `match/6`. I executed the actual clause bodies on a 500,000-element proper-list literal: it completed successfully, produced a correctly terminated `[{'|',[],[500000,[tailmarker]]}]` tail, and did **not** blow the BEAM stack — confirming the body recursion scales to source-proportional depth with no amplification.
- Axes examined: termination (strictly-decreasing `Tail`), throw containment, improper-list tail (`is_list(Tail)` guard), measured stack depth for an N-element literal (executed at N=500k) | Not examined: none.
- Basis: verified.
- What I ran: `axtest:sa/3` on `lists:seq(1,500000)` with tail `[tailmarker]` (completed, correct tail, no overflow).

- **`length/1` on AST argument lists** — `erl_to_ex/3` line 48, `match/6` line 302, `is_always_string/1` line 361 (crash / robustness class)
- Reason: `Args` is compiler-internal AST reached from `elixir_dispatch`/`elixir_expand`; a proper list by construction. An improper list would raise `badarg` — a self-inflicted compile-time crash in the caller's process with no cross-boundary effect.
- Axes examined: proper vs improper list (a bare improper list `[a|b]` given to `length/1` raises `badarg`, confirmed as the standard `length/1` contract), integer-arity vs list clause of `erl_to_ex` | Not examined: whether a hand-crafted macro could inject an improper list into the args slot **end-to-end through `elixir_expand`/`elixir_dispatch`** — this axis requires pushing a value through the whole compiler front-end, which is not an axis I can exercise from this file in isolation; reading those neighbour modules would not clear it (it tells me what they build, not which inputs reach `length/1` improper). Execution not attempted.
- Basis: asserted.
- Depends on: `elixir_expand.erl`/`elixir_dispatch.erl` — deliver well-formed AST (proper arg lists) to these entry points. Even if the axis were reachable, the outcome is a self-inflicted `badarg` in the caller's own process, not a security defect.

- **`is_always_string/1,3` interpolation optimization** — lines 360–371 (silent-capability / round-trip class)
- Reason: the allow-list (`Enum.join`, `Enum.map_join`, `Kernel.inspect`, `Macro.to_string`, `String.Chars.to_string`, `Path.join`) are all functions that return binaries, so skipping the `String.Chars.to_string` wrap in interpolation is sound; a hypothetical wrong `true` only affects the author's own runtime result (a malformed-binary/`ArgumentError`), crossing no trust boundary. `Module` is matched against fully-resolved `'Elixir.*'` atoms, not caller-aliasable.
- Axes examined: alias-resolution of `Module` (matched against literal `'Elixir.*'` atoms), non-`.` AST fallthrough to `is_binary/1` (line 363) | Not examined: whether any listed function has an arity/variant returning non-binary — this axis is a property of the `.ex` standard-library sources (`Enum`/`Kernel`/`Macro`/`String.Chars`/`Path`), not of any module in this folder, and is not exercisable as an input-comparison axis from `elixir_rewrite.erl`; execution not attempted.
- Basis: asserted.
- Depends on: `String`/`Enum`/`Path`/`Kernel`/`Macro` stdlib — `join`/`map_join`/`inspect`/`to_string` return binaries for all arities. Regardless of the answer, this is a correctness/robustness concern with no trust boundary (a wrong `true` yields the author's own runtime error).

- **Arithmetic folds & index math** — `rewrite/5` lines 254–255, `inner_rewrite` `element`/`setelement` index `Index - 1` lines 269–276, `increment/2` lines 287–288 (unclassified robustness)
- Reason: every arithmetic site is guarded by `is_number/1`, so no `badarith`; Erlang integers are arbitrary-precision (no overflow); the results are AST literals, not executed here.
- Axes examined: integer vs float operand, `is_number` guard coverage, overflow (bignum) | Not examined: none.
- Basis: verified.

### Did I verify this?

- **Rewrite-table targets (atoms / ports / spawn)** — `verified`. Lines 62–241, 260–283: `inner_inline`/`inner_rewrite` clause bodies only construct and return tuples; no call to the named Erlang functions. `Not examined: none`.
- **`Receiver:'__info__'(macros)` dispatch** — **now `verified`** (upgraded from `asserted`). Executed the dispatch across all four axes: non-atom → badarg → caught; non-existent module → undef → caught; Erlang module lacking `__info__` → undef → caught; side-effecting author `__info__` → side effect runs but is author-controlled code, and the raising case is caught. Reading `elixir_expand.erl` was **not** the path to verification (it could not clear a behavioural axis); I ran the behaviour. `Not examined: none`.
- **`io_lib:format` data-derived format string** — `verified`. Ran the real `format_error/1` with adversarial `:"a~w~w~w"` (no crash, literal render). `Not examined: none`.
- **`allowed_guard/2`** — `verified`. Executed the full `erl_internal:guard_bif/2` and `guard_op/2` true-set enumeration; no side-effecting entry; `send`/`!`/`++`/`--` excluded. Noted `is_record/1` correctness curiosity (see Risks). `Not examined: none`.
- **`static_append/3` depth** — **now `verified`** (upgraded from `asserted`). Ran the clause bodies on a 500k-element literal: completes, correct terminated tail, no stack overflow; depth is source-proportional (no amplification). This is the axis the first pass left un-tried; it has now been executed. `Not examined: none`.
- **Arithmetic folds** — `verified`. `is_number/1` guards read; bignum semantics language-guaranteed. `Not examined: none`.
- **`length/1` on AST arg lists** — stays `asserted`. The end-to-end reachability axis (can a macro inject an improper list all the way into the args slot) requires driving `elixir_expand`/`elixir_dispatch`, which is not an input-comparison axis exercisable from this file; reading those modules cannot clear it. Execution not attempted; kept `Depends on: elixir_expand.erl`/`elixir_dispatch.erl`. Even if reachable, the outcome is a self-inflicted `badarg` with no cross-boundary effect.
- **`is_always_string/1,3`** — stays `asserted`. The un-tried axis (does any listed function's arity return non-binary) is a property of the `.ex` stdlib, not of this folder, and is not an executable axis here. Kept `Depends on:` the `String`/`Enum`/`Path`/`Kernel`/`Macro` return-type contract. Correctness/robustness only — no trust boundary regardless of the answer.

### Dropped in re-review

No prior findings dropped — the first pass reported **No findings**, and that verdict is unchanged. Status changes this pass, all backed by executed evidence:
- **`Receiver:'__info__'(macros)` dispatch**: `asserted → verified` (Confidence: high) — every axis, including the side-effecting-`__info__` axis, tried by execution; behaviour is author-controlled compile-time code, out of scope per the folder threat model.
- **`static_append/3` depth**: `asserted → verified` (Confidence: high) — the previously-un-tried "measured stack limit for an N-element literal" axis executed at N=500k (completed, no overflow).
- **`length/1` on AST arg lists** and **`is_always_string/1,3`**: deliberately **kept `asserted`** (not upgraded to `verified`), because their remaining `Not examined` axes are not exercisable as input-comparison axes from this file and reading neighbours cannot clear them — per the pass rules, an un-executed axis stays `asserted`. Neither is a security finding (both are self-inflicted, no-trust-boundary robustness/correctness notes), so neither is upgraded to a low-confidence finding.
