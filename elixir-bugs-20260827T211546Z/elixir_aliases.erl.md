# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_aliases.erl
<!-- sha256: a03900c11f93f69518df1bdf1c0d3c1ab39f4e6bd1ec3fb5705c75e8bb656698 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_aliases.erl
<!-- sha256: a03900c11f93f69518df1bdf1c0d3c1ab39f4e6bd1ec3fb5705c75e8bb656698 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

# Review: `elixir_aliases.erl`

## 1. Summary

The module faithfully mirrors upstream Elixir alias/concat/expand logic. The first-pass report dismissed the `do_concat` charlist gap as unreachable, resting that dismissal on an *unread* cross-file typespec in `lib/module.ex` (the "outside this file" hedge). On re-review I (a) executed the charlist axis and confirmed `do_concat` raises `function_clause` on a list-of-integers element, and (b) read the two in-folder callers of `concat/1` (`elixir_expand.erl:1041`, `elixir_expand.erl:1088`), which feed it `__aliases__` segment lists derived from user-macro AST rather than from `Module.concat/1`. Because the previous dismissal's "not examined" axis is a *reachability* claim that reading a neighbour cannot clear, and the one concrete gate cited was never inspected, I am upgrading it to a low-confidence finding rather than leaving it `asserted`. All other classes remain `verified`.

## 2. Bugfixes

### Finding 1 — `do_concat/1` raises `function_clause` on a charlist segment; in-file comment says lists are accepted
**Upgraded from hedged assumption in first pass** (was: "Crash on stray input" + "Documented behaviour mismatch", both `asserted` on `Depends on: lib/module.ex`).

**Confidence: Low.**

- **Sink class:** Crash on stray / unexpected input (also a Documented-behaviour mismatch).
- **What is confirmed (executed):** `do_concat/2` (lines 187–194) has clauses only for `nil`, `is_atom`, `is_binary`, `[]`. A list element (charlist, e.g. `"Bar"`/`~c"Bar"` = list of integers) matches none and raises `function_clause`. I reproduced this against a byte-for-byte copy of `concat/1`+`do_concat/2`:
  - `concat([foo, bar]) => 'Elixir.foo.bar'`
  - `concat([<<"Foo">>, <<"Bar">>]) => 'Elixir.Foo.Bar'`
  - `concat(["Foo"]) => function_clause at do_concat/2`
  - `concat([foo, "Bar"]) => function_clause at do_concat/2`
- **Documented-behaviour half (verified in-file, no cross-file dependency):** the comment at lines 172–173 states the function "Receives a list of atoms, binaries **or lists** representing modules." The code crashes on the "or lists" case. This mismatch is fully contained in this file and is now confirmed, not asserted.
- **Reachability (why Low, not dismissed):** the in-folder callers I read — `elixir_expand.erl:1041` (`no_alias_expansion({'__aliases__', _, [H|T]}) -> concat([H|T])`) and `elixir_expand.erl:1088` (`concat([EHead | Tail])`) — pass `__aliases__` AST segment lists. Those segments are atoms in the normal case, but `__aliases__` node lists are user-macro-constructible; a macro emitting a charlist segment would reach `do_concat` and crash the compile job for that file. I did **not** confirm a concrete macro path that reaches here without earlier segment validation, so I cannot assert exploitability — hence Low. What I can rule out is the previous basis for dismissal: the safety does *not* come from anything in this file, and it does *not* come from the callers I read; it would have to come entirely from `Module.concat/1`'s typespec in `lib/module.ex`, which is not a callgraph neighbour and which I did not (and per the re-review rules, cannot by reading) use to clear the reachability axis.
- **Failure scenario:** a macro or metaprogramming path produces an `__aliases__` (or a direct `Module.concat/1` call from Erlang-side code) whose segment list contains a charlist; the compiler front-end aborts that file's compilation with an opaque `function_clause` in `elixir_aliases:do_concat/2` instead of a formatted diagnostic.
- **Fix direction:** either add an `is_list(H)` clause to `do_concat/2` that treats a charlist as `list_to_binary`, or correct the lines 172–173 comment to state only `atom | binary` are accepted (matching `Module.concat/1`'s intended contract).

## 3. Risks / caveats

- **`expand_as/3` A–Z guard (lines 44/58).** The `FirstLetter >= $A, FirstLetter =< $Z` guard is stricter than a bare `"Elixir." ++ Rest` prefix test: `as: :"Elixir.foo"` (lowercase) yields `not_alias` rather than being accepted. This is a tightening, not a regression — such lowercase values are not valid aliases — but it changes the error class for that edge input. (Unchanged from first pass; `verified` by inspection.)

### Ruled out

**Operator / precedence typo**\
Reason: guards are simple single-condition comparisons (`FirstLetter >= $A, FirstLetter =< $Z`, `is_atom`, `H /= nil`); the only comma-guard (lines 44/58) is a range test whose `,`-conjunction is correct, and there are no `;`-alternation guards to mis-partition.\
Axes examined: uppercase-first alias, lowercase-first, empty remainder, boolean-as-atom | Not examined: none\
Basis: verified

**Missing clause / catch-all**\
Reason: every `case` (alias/6 result switch, expand/4 meta switch, lookup/3, ensure_loaded) has an explicit fall-through or covers all shapes the in-file producers emit.\
Axes examined: `as` absent/false/true/atom/non-atom, meta with/without `alias` and `counter` keys, lookup hit/miss/counter-mismatch | Not examined: none\
Basis: verified

**Wrong atom / wrong symbol**\
Reason: error atoms produced (`invalid_alias_for_as`, `nested_alias`, `not_alias`, `invalid_alias_module`, `circular_module`, etc.) each have a matching `format_error/1` clause; no key/atom drift.\
Axes examined: each error tuple vs each format_error head | Not examined: none\
Basis: verified

**State / accumulator threading error**\
Reason: `alias/6` and `expand/*` thread the env `E`/accumulator immutably and return the updated map/atom; `last/2` and `do_concat/2` accumulators build in correct order (hand-traced "Elixir.Foo.Bar" and confirmed by executing `concat([foo,bar]) => 'Elixir.foo.bar'`).\
Axes examined: single-segment, multi-segment, empty-tail alias expansion | Not examined: none\
Basis: verified

**Off-by-one / wrong arithmetic**\
Reason: no index or length arithmetic; `last/2` splits on the `$.` delimiter, not on a computed offset.\
Axes examined: trailing segment extraction across 1/2/3-segment names | Not examined: none\
Basis: verified

**Contract violation**\
Reason: `alias/6` always returns one of `{ok,_,_}` / `{error,_}`; `expand/4` returns list-or-atom per its comment; `concat`/`safe_concat` return atoms.\
Axes examined: all documented return shapes per export | Not examined: none\
Basis: verified

**Crash on stray / unexpected input**\
Reason: PROMOTED to Finding 1 above. The charlist axis was executed (`concat(["Foo"])` and `concat([foo,"Bar"])` both raise `function_clause` in `do_concat/2`), and the in-folder callers `elixir_expand.erl:1041`/`1088` feed macro-constructible `__aliases__` segment lists, so the "unreachable via documented API" dismissal — which depended on an unread `lib/module.ex` typespec — no longer holds as a dismissal.\
Axes examined: empty list, nil element, empty atom, non-"Elixir" binary, `<<"ElixirFoo">>`, bare charlist element (executed) | Not examined: none\
Basis: promoted to finding (see Finding 1)

**Resource leak on error path**\
Reason: module allocates no files/ETS/processes/monitors; `wait_for_module` delegates to `Kernel.ErrorHandler` (another module) and returns a plain atom.\
Axes examined: n/a (no resources acquired here) | Not examined: none\
Basis: verified

**Silent corruption**\
Reason: `do_concat`/`to_partial`/`last` outputs traced to correct binaries for representative inputs (executed `concat([<<"Foo">>,<<"Bar">>]) => 'Elixir.Foo.Bar'`); `lookup/3` counter-mismatch correctly returns not-found rather than a wrong alias.\
Axes examined: macro-alias counter match/mismatch, regular-alias atom value, "Elixir." prefix stripping | Not examined: none\
Basis: verified

**Infinite loop / hang**\
Reason: all recursion (`do_concat`, `last`, `expand`) consumes a list head per step toward `[]`; no `receive`/`gen_server` loop in this module.\
Axes examined: list reduction termination | Not examined: none\
Basis: verified

**Quadratic-where-linear / Repeated work**\
Reason: accumulators use binary append into a growing `Acc` (`do_concat/2`) which is amortized-linear on the BEAM, and `[H|Acc]` prepend in `last/2`; no `++` in a loop, no per-call pattern recompilation.\
Axes examined: long alias chains | Not examined: none\
Basis: verified

**Dead / unreachable code**\
Reason: each clause is reachable — traced triggers for every `expand_as` and `do_concat` head.\
Axes examined: all clause heads | Not examined: none\
Basis: verified

**Documented behaviour mismatch**\
Reason: PROMOTED to Finding 1 above. The lines 172–173 "or lists" comment vs the `do_concat/2` crash-on-charlist is now confirmed in-file by execution, not asserted. This is no longer a bare caveat.\
Axes examined: comment vs clause coverage for concat, charlist element behaviour (executed) | Not examined: none\
Basis: promoted to finding (see Finding 1)

**Race condition / shared mutable state**\
Reason: module is stateless library code run in the caller's process; env `E` is an immutable map, no ETS/process state mutated here.\
Axes examined: n/a | Not examined: none\
Basis: verified

**Wrong scope / order of operations**\
Reason: trace side-effects (lines 17, 30, 123) occur after the successful branch is selected and before returning the updated env, matching intent.\
Axes examined: alias store vs trace ordering | Not examined: none\
Basis: verified

**Encoding / unicode mishandling**\
Reason: `concat`/`safe_concat` use `utf8` consistently for both `atom_to_binary` and `binary_to_atom`; no latin1 default.\
Axes examined: utf8 round-trip on atom↔binary | Not examined: none\
Basis: verified

**Unclassified**\
Reason: no residual file-specific anomaly after reading all 277 lines.\
Axes examined: full-file read | Not examined: none\
Basis: verified

### Did I verify this?

Every class above is `verified` with `Not examined: none` **except** the two classes I promoted to Finding 1 (Crash on stray input; Documented behaviour mismatch). For those two:

- The **in-file** halves are now `verified` by execution: I compiled a byte-for-byte copy of `concat/1` + `do_concat/2` + `to_partial/1` and ran it. `concat(["Foo"])` and `concat([foo, "Bar"])` both raise `function_clause` in `do_concat/2`; `concat([foo,bar])` and `concat([<<"Foo">>,<<"Bar">>])` succeed. This removes the prior "Not examined: bare charlist element" gap by *trying* the axis, not by reading a neighbour.
- The **reachability** half remains uncertain and is the reason the finding is Low, not dismissed and not a full bug. I read the two in-folder callers (`elixir_expand.erl:1041`, `1088`); both pass `__aliases__` segment lists, which are macro-constructible, so I cannot claim unreachability. The prior report's sole basis for calling it unreachable — the `Module.concat/1` typespec in `lib/module.ex` — is not a callgraph neighbour and, per the re-review rules, could not clear the reachability axis even if read. I therefore did not re-assert the dismissal.

### Dropped in re-review

No prior findings dropped. The two prior `asserted` **dismissals** (Crash on stray input; Documented behaviour mismatch) were not dropped but *upgraded* to a single low-confidence finding (Finding 1), as recorded above. The prior Risk/caveat about the `expand_as` A–Z guard is retained unchanged.
