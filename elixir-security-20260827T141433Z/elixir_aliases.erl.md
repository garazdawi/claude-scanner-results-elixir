# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_aliases.erl
<!-- sha256: a03900c11f93f69518df1bdf1c0d3c1ab39f4e6bd1ec3fb5705c75e8bb656698 -->

_Analysed on 2026-08-27T14:34:06Z_

Based on my review, I have read the file, traced every sink through its in-folder callers (`elixir_expand.erl`, `elixir_quote.erl`, `elixir_compiler.erl`, etc.), and reproduced the key behaviors in Erlang. Here is my report.

---

## Summary

`elixir_aliases.erl` is the compiler's module-alias resolver and `Module.concat` backend; its only security-relevant sinks are atom-creating conversions (`binary_to_atom`/`list_to_atom`) whose atom generation is the documented, bounded purpose of the API, and I found no reportable defect.

## Security

No findings.

Every dangerous-shaped sink in this file resolves to either (a) atom creation that is the *documented contract* of the function with a safe sibling (`safe_concat`) provided in the same module, (b) a value bounded by the size of source the caller already chose to compile, or (c) constant-format-string display logic. Details are enumerated under Ruled out.

## Risks / caveats

- **`Module.concat/1,2` is an atom-creating public API by design.** `concat/1` (line 175, `binary_to_atom/2`) will convert an unbounded set of distinct caller-supplied binaries into atoms, so a *library consumer* who forwards untrusted strings into `Module.concat` (a real anti-pattern in web routing) can exhaust the atom table and crash the BEAM. This is not a defect in `elixir_aliases.erl`: the function is documented to do exactly this, and `safe_concat/1` (`binary_to_existing_atom/2`, verified to raise `badarg` on a non-existing atom) is the documented safe replacement. The bug, if any, lives in such a caller, not here.
- **The `.ex` wrappers (`Module.concat`, `Module.safe_concat`) are not in this checkout** — `src` is a symlink to a single folder of `.erl` files; `lib/elixir/lib/module.ex` was unavailable. My characterization of their documented contract relies on the well-known Elixir public API, not on reading that file. If those wrappers were to silently route untrusted input into `concat` without the documented warning, the risk assessment would shift to the wrapper.
- **Programmatically constructed atoms bypass the tokenizer's identifier-security checks** (mixed-script/NFC/bidi). `concat(["Elixir.Аdmin"])` with a Cyrillic homoglyph yields a confusable module atom, because UTS #39/UAX #31 enforcement lives only on the *source-parsed* identifier path in `elixir_tokenizer`, not on programmatic `Module.concat`. This is an accepted, long-standing boundary of the language, not a regression in this file.

### Ruled out

**`concat/1` → `binary_to_atom(do_concat(Args), utf8)` (line 175) — atom-table exhaustion (CWE-770).**\
Reason: atom creation is the documented purpose of `concat`; the module ships `safe_concat` (`binary_to_existing_atom`) as the safe form for untrusted input, and on the automatic compilation path (`elixir_expand:expand_aliases/4` → `concat`, read at elixir_expand.erl:1088) the atoms are a bounded function of source the caller already chose to compile — the tokenizer already atomises every identifier, so `concat` adds only a constant factor and no new capability.\
Axes examined: element cardinality/distinctness, atom-vs-binary element types, nil element, leading-dot segment, `"Elixir."`-prefixed vs bare segment | Not examined: multi-megabyte single-segment binary (large-atom cost), non-UTF8 bytes reaching `binary_to_atom` from a non-source caller.\
Basis: asserted.\
Depends on: `lib/elixir/lib/module.ex` — `Module.concat/1,2` documents unsafe atom creation and directs untrusted input to `safe_concat`; `elixir_tokenizer.erl` — identifiers in compiled source are already atomised there, bounding the compile-path atom count.

**`safe_concat/1` → `binary_to_existing_atom(do_concat(Args), utf8)` (line 176) — atom exhaustion.**\
Reason: uses the existing-atom variant, which cannot create a new atom; verified to raise `badarg` when the target atom does not already exist.\
Axes examined: existing vs non-existing atom, atom/binary element mix | Not examined: none.\
Basis: verified (`ran erlc t_aliases.erl && erl … t_aliases:run()` → `safe_concat nonexisting: {'EXIT',{badarg,[{erlang,binary_to_existing_atom,[<<"Elixir.NoSuchMod_ZZZ.Nope_QQQ">>,utf8]…}}`).

**`expand/5` → `list_to_atom("Elixir." ++ atom_to_list(H))` (line 113) — atom creation.**\
Reason: `H` is already an atom (guaranteed by the guard clauses at lines 96–100 that divert non-atom heads), so no *new* segment atom is minted; only the `Elixir.`-prefixed lookup atom is created, one per distinct alias head in source — bounded by source size, same posture as `concat`.\
Axes examined: atom-vs-non-atom head (diverted upstream), head cardinality | Not examined: adversarial head length.\
Basis: verified (read /cscan/0/elixir_aliases.erl:96-128; head type enforced by clauses 96/99).

**`expand_as(false, true, Ref)` → `list_to_atom("Elixir." ++ Last)` (line 60) — atom creation.**\
Reason: `Ref` is a pre-existing module atom; the derived last-segment atom is bounded by the number of distinct aliased modules in source. Guard requires `Elixir.`-prefix + uppercase first letter, else `{error, invalid_alias_module}`.\
Axes examined: `Elixir.`-prefixed vs bare, uppercase vs lowercase first letter, empty tail (`'Elixir.'`), nested name | Not examined: none.\
Basis: verified (`ran` — `expand_as lower` → `{error,{invalid_alias_for_as,not_alias,'Elixir.foo'}}`; simple/nested cases as expected).

**`expand_as/3` `:as` validation predicate (lines 42–53) — validation (wrong-answer risk).**\
Reason: the predicate's job is to accept only a simple (single-segment) `Elixir.`-prefixed alias; `string:tokens(Rest, ".")` length-1 check correctly rejects nested aliases, and non-alias / boolean / lowercase / `nil` values fall through to `{error, …}`. A wrong "accept" here only stores an alias mapping — no code execution or path.\
Axes examined: simple alias, nested alias, lowercase start, `nil`, boolean, non-atom `:as` | Not examined: `..`-collapsing in `string:tokens` (cosmetic; source cannot emit `..` in an alias atom).\
Basis: verified (`ran` — nested → `nested_alias`; bool/nil/lower → `not_alias`).

**`ensure_loaded/3` → `code:ensure_loaded(Module)` (line 146) — code loading.**\
Reason: `Module` is a module atom derived from source aliases; loading a referenced module (running its `on_load`) is inherent to compiling any code that references it, so it grants the source author nothing beyond what referencing the module already does. Self-reference is rejected (`circular_module`, line 143); `Kernel` is skipped for bootstrap only.\
Axes examined: self-module, `Kernel`, loadable vs unloadable module | Not examined: none.\
Basis: verified (read /cscan/0/elixir_aliases.erl:141-164; callers elixir_expand.erl:122,155 pass source-derived `ERef`).

**`wait_for_module/3` → `'Elixir.Kernel.ErrorHandler':ensure_compiled/4` (line 169).**\
Reason: internal compiler-coordination call gated on `erlang:get(elixir_compiler_info)`; not a documented external entry point and no attacker-controlled term beyond the already-in-scope `Module`.\
Axes examined: compiler-info present vs absent | Not examined: none (internal).\
Basis: verified (read lines 166-170).

**`format_error/1` → `io_lib:format(…)` (lines 211–276) — format-string / terminal-escape injection.**\
Reason: every `io_lib:format` call uses a compile-time **literal** format string; the interpolated values are module names / `:as` values rendered via `'Elixir.Macro':to_string/1` for a developer-facing compile diagnostic — no attacker-chosen format directives, and source identifiers have bidi/control already rejected by the tokenizer.\
Axes examined: format-string constancy, `~ts`/`~s` arg source | Not examined: control-byte content of a *programmatically* built module atom reaching an error message (out-of-band from source; requires prior code execution).\
Basis: verified (read lines 211-276; all format strings literal).

**`inspect/1` → `'Elixir.Macro':inspect_atom(literal, Atom)` (line 13) — human-in-the-loop formatting.**\
Reason: renders a module atom for diagnostics; escaping/quoting of atoms is `Macro`'s responsibility, and the value is not used for any security decision in this file. Bootstrap branch just does `atom_to_binary`.\
Axes examined: bootstrap vs non-bootstrap, atom input | Not examined: `inspect_atom` escaping behaviour (lives in `Macro`, out of file).\
Basis: asserted.\
Depends on: `Elixir.Macro` (`lib/elixir/lib/macro.ex`) — `inspect_atom(literal, _)` escapes atoms requiring quoting.

**`do_concat/1,2` binary accumulation (lines 178–194) — resource consumption / quadratic append.**\
Reason: `<<Acc/binary, …>>` is used linearly, triggering the runtime's append optimisation (amortised O(n)); the argument list is a bounded list of module segments, so no super-linear blowup. `nil`-head and leading-dot cases handled without crashing.\
Axes examined: nil head, leading-dot segment, binary vs atom elements, `"Elixir"`/`"Elixir."` prefixes | Not examined: pathologically long argument list (still linear).\
Basis: verified (`ran` — `concat nil-head` → `'Elixir.Foo'`; `concat leading-dot` → `'Elixir.Foo.Bar'`).

**Shared mutable state — none.**\
Reason: `alias/6`, `require/5`, `expand/*` operate purely on the functional `E` env map returned to the caller; no ETS, process-dictionary, or application-env writes occur in this file. `elixir_config:is_bootstrap()` (line 11) is a read-only global read.\
Axes examined: ETS writes, process-dict writes, app-env writes, cross-caller poisoning | Not examined: none.\
Basis: verified (read whole file; no write sinks present).

### Did I verify this?

- **`concat/1` atom exhaustion — asserted.** The atom-creation *behaviour* is verified by run (`concat bin+bin` → `'Elixir.Foo.Bar'`), but the *dismissal* rests on two out-of-file contracts I could not read here: (1) `Module.concat`'s documented "unsafe atom, use `safe_concat`" warning in `lib/elixir/lib/module.ex`, and (2) the tokenizer being the primary atom source that bounds the compile path. Both are standard, long-documented Elixir behaviour, but they live outside this checkout — kept `asserted` with the `Depends on:` lines above so the cross-file pass can confirm them. The untried axes (multi-MB single segment, non-UTF8 bytes from a non-source caller) would at most change a bounded DoS into a slightly larger bounded DoS, not create a new capability.
- **`safe_concat/1`, `expand/5`, `expand_as` (both clauses + validation), `ensure_loaded/3`, `wait_for_module/3`, `format_error/1`, `do_concat`, shared-state — verified.** Each was read in full and, where behavioural, exercised in Erlang (`ran t_aliases:run()`); `Not examined` is `none` or names only axes that are unreachable from a documented source path (e.g. `..` in an alias atom, which the tokenizer cannot emit).
- **`inspect/1` — asserted.** Correctly depends on `Macro.inspect_atom/2`'s escaping, which is out of file; kept `asserted` with the `Depends on:` line rather than claimed as verified.

