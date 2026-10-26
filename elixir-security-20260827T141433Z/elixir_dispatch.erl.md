# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_dispatch.erl
<!-- sha256: f51f579e64a84c079af268323cd7e7677ed41ec04fa40b23ce6ba479c58bb378 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_dispatch.erl
<!-- sha256: f51f579e64a84c079af268323cd7e7677ed41ec04fa40b23ce6ba479c58bb378 -->

_Re-reviewed on 2026-08-28 (hedge-resolution pass #2 over the first-pass report)._

The flagged hedge — *"did not exhaustively audit `elixir_expand`/`elixir_def`/`elixir_import`/`elixir_quote`"* — is discharged here, but **not** the way the intervening pass claimed. That pass read the reached callees and then upgraded eight ruled-out entries to `Basis — verified` with `Not examined: none`. Reading a guard tells you what it does on the inputs you imagined, not which inputs make it disagree with the caller's assumption; under this pass's rule a read does not clear an axis. I re-ran only the axis I could actually execute (atom-table growth) and downgraded every entry whose dismissal rested on a neighbour read or on un-executed data-flow reachability back to `asserted`, restoring the untried axes to `Not examined`. This is the safe direction: a later pass re-examines an `asserted` dismissal; it never re-examines a wrongly-`verified` one.

Conclusion is unchanged in substance: **no findings.** What changed is confidence bookkeeping — most dismissals are now honestly `asserted`, and exactly one axis (atom exhaustion via `macro_name`) is `verified` with cited executed output.

---

## Summary

`elixir_dispatch.erl` is compile-time import/require dispatch over already-parsed AST and the compiler environment. Every security-relevant sink is one of: (a) code execution inherent to macro expansion, which the folder threat model places out of scope (compiling untrusted source *is* granting compile-time execution); (b) module introspection/loading of modules the source already references; or (c) atom creation bounded to already-interned macro names (the only sink for which I have executed evidence). **No findings.**

## Security

No findings. (No finding-level Confidence field applies, as there are no findings; the required Confidence field would appear on any finding, and there are none.)

The Unicode/tokenizer standards obligations in the folder checklist (BIDI rejection, `?break` newline handling, NFC, mixed-script, `\u`/`\x` escapes, UTF-8 decoding, grapheme segmentation) and the RFC compliance clauses (IDENT_XID_SYNTAX, IDENT_NFC_FORM, IDENT_MIXED_SCRIPT_REJECT, ALIAS_ASCII_ONLY, the UAX #24 script-property clauses, NFC conformance, BIDI reordering) do **not** apply to this file. `elixir_dispatch.erl` performs no tokenization, identifier validation, escape decoding, script/NFC checks, or byte-level source handling. It receives atoms already produced and vetted by `elixir_tokenizer`/`elixir_parser` and dispatches calls against the environment's import/macro ordsets. Those obligations live in `elixir_tokenizer.erl`, `elixir_tokenizer.hrl`, `elixir_interpolation.erl`, and `elixir_utils.erl`.

### Ruled out

**`apply(Fun, [Caller | Args])` — macro invocation (line 263, `expand_macro_fun/7`)**
- Reason — Invoking `Fun` runs a macro at compile time. The dismissal rests primarily on folder-threat-model policy: running the macros of the code being compiled is the documented contract of compilation, and sandboxing eval/compile is out of scope. Independently, `Fun` is intended to be a validated macro — either a local from `elixir_def:local_for/5` restricted to `[defmacro, defmacrop]`, or `fun Receiver:'MACRO-Name'/Arity+1` gated by `is_macro/4`.
- Neighbour context (does NOT clear an axis) — I re-read `elixir_def:fun_for/6` (`elixir_def.erl:34-55`): it looks up `{def, Tuple}` in the module's ETS Set, and returns `false` unless a definition exists *and* `lists:member(Kind, Kinds)` holds, the whole body wrapped `catch _:_ -> false`. This is consistent with "no fun for a non-macro/non-existent def." But reading the guard does not tell me which crafted `Meta`/`Module`/`Name` triple could make `local_for`'s ETS lookup return a fun of an unexpected kind under a bootstrap/self-referential module; that is the untried `unvalidated-fun` axis, and I did not execute it. It stays on `Not examined`.
- Axes examined: is-it-in-scope (folder threat model: compile == run macros) | Not examined: unvalidated-fun (whether `local_for` can yield a non-`defmacro*` fun for adversarial inputs — reading `fun_for` describes the guard but does not exercise it), `Caller`/arity injection into `apply`
- Basis — asserted (dismissal holds on policy; the fun-validation axis is not execution-backed) — **Downgraded from the intervening pass's `verified`; reason: its `verified` rested on reading `fun_for`, which this pass's rule does not accept as clearing the axis.**

**`elixir_expand:expand(ToExpand, S, E)` and `elixir_quote:linify_with_context_counter/3` — recursive processing of macro output (lines 276-277, `expand_quoted/7`)**
- Reason — Expands the AST a macro returned; a hostile macro can return huge/self-recursive AST, but that is self-inflicted compile-time DoS by untrusted-macro code already running by design, bounded to the compiling caller's own process per the folder threat model ("a parse crash kills only the caller").
- Neighbour context (does NOT clear an axis) — the intervening pass read `elixir_quote:do_linify/4` and asserted it mints no atoms. Even granting that read, "no atom sink in `linify`" is a claim about the callee's code on the inputs imagined, not an executed differential over adversarial metadata; I did not run AST through `linify`/`expand` to observe atom-table or stack behaviour. Both the atom-creation and recursion-depth axes therefore return to `Not examined`.
- Axes examined: is-it-in-scope (self-DoS by already-running macro code, per threat model) | Not examined: atom-creation inside `linify`/`expand` (not executed), recursion-depth / AST-size blowup (not executed)
- Basis — asserted — **Downgraded from `verified`; reason: the no-atom-sink claim was read, not run.**

**`code:ensure_loaded(Receiver)` — module load during deprecation check (line 389, `get_deprecations/1`)**
- Reason — Loading `Receiver` (a module the source explicitly references in a remote call) is intrinsic to compiling code that calls into it; running an `on_load` is the referenced module's own code. All three gates are readable in *this* file: `check_deprecated/6` short-circuits `erlang`, `elixir_def`, `elixir_module`, `?kernel` (lines 420-423), and only reaches `get_deprecations` when `(?key(E, function) == nil) or (Kind == macro)` (line 435). No path lets an attacker name a module the source does not already reference.
- Axes examined: guard reachability (`function==nil` / `Kind==macro`, in-file), short-circuit list (in-file) | Not examined: on-load side-effect payload (out of scope — the referenced module's own code, per threat model), whether an attacker can route an *unreferenced* module here (not executed; argued from in-file control flow only)
- Basis — asserted (dismissal rests on in-file control flow plus threat-model policy; not execution-verified) — **Downgraded from `verified`: this pass reserves `verified` for executed axes only.**

**`Receiver:'__info__'(macros)` / `Receiver:'__info__'(Key)` — introspection calls (lines 380, 398, `is_macro/4`, `get_info/2`)**
- Reason — Calls the standard `__info__/1` reflection function on a source-referenced module; both are wrapped `try … catch error:_ -> false | []` (lines 380-384, 397-401), and `get_info/2` guards with `erlang:function_exported/3` first (line 395). A missing/ill-behaved `__info__` degrades to "not a macro / no deprecation" rather than crashing.
- Axes examined: missing-function crash (caught, in-file), non-list return (`is_element`/`keyfind` tolerate it, in-file) | Not examined: attacker-chosen receiver escalation (argued from threat-model policy — module is source-referenced — not executed)
- Basis — asserted — **Downgraded from `verified`: the crash-safety branches are readable in-file, but the "no new capability" conclusion leans on policy, not execution.**

**`elixir_utils:macro_name(Name)` → `list_to_atom("MACRO-" ++ ...)` — atom creation (lines 256, 267, 280)**
- Reason — `macro_name/1` is `list_to_atom("MACRO-" ++ atom_to_list(Macro))` (confirmed at `elixir_utils.erl:27-28`), a single deterministic derivation. The security question is whether a call site can drive it with *distinct novel* names (table growth) rather than re-deriving already-interned macro names.
- Executed evidence (this session) — I re-ran the differential on the local OTP toolchain:
  - Re-deriving an atom that is *already interned*, 100 000 times: `delta = 0` (command: pre-`list_to_atom("MACRO-existing")`, then 100 000 re-derivations; `system_info(atom_count)` before/after → 0).
  - Minting 5 000 *distinct* novel `"MACRO-novel-N"` names: `delta = 5000` (one atom per distinct name).\
  This confirms the sink grows the table only with distinct novel inputs; re-derivation of an interned name is free.
- Un-executed axis — Whether this file's call sites can *supply* distinct novel names is a data-flow reachability question. Lines 256/267/280 are reached after a `{macro, Receiver}` dispatch out of the environment's macro ordset, or after `is_macro/4`, or inside error handlers on an already-built `Fun`. Arguing that `Name` is thereby bounded to source-declared macro names requires tracing how `elixir_import` populates those ordsets — which I read the *shape* of, not executed. So the reachability axis is `Not examined`.
- Axes examined: table-growth-per-distinct-vs-repeated-name (executed — see above) | Not examined: whether the call sites can supply attacker-chosen distinct names (data-flow reachability through `elixir_import` ordset population — read, not executed), forged-`{import,Receiver}` metadata path
- Basis — asserted overall; the atom-table-growth axis specifically is **verified (executed this session)**. — **Downgraded from blanket `verified`: the reachability sub-axis is not execution-backed even though the growth differential is.**

**`is_import/2` → `{import, Receiver}` forcing `expand_require(true, …)` — hygiene-metadata-driven dispatch (lines 331–343, 218–219)**
- Reason — An AST node's `Meta` carrying `{imports,[{Arity,Receiver}]}` + `{context,_}` short-circuits import resolution to `Receiver` with `Required=true`. This metadata is internal `elixir_quote` hygiene state; reaching it with forged metadata requires feeding a hand-built AST to `eval_quoted`/`expand`, documented as a caller bug. The effect is invoking a macro that genuinely exists in `Receiver`, reachable equally via `require Receiver`.
- Un-executed axis — the claim "surface syntax cannot set this meta" is an assertion about `elixir_tokenizer`/`elixir_parser`/`elixir_quote` behaviour I did not exercise (I did not attempt to drive crafted source through the parser to see whether `imports`/`context` meta can be produced). It stays `Not examined`.
- Axes examined: existence gating via `is_macro/4` on the resolved `Receiver` (in-file, lines 242-248) | Not examined: whether surface syntax / a crafted AST can set `{imports,…}`+`{context,…}` meta (not executed against the parser), require-bypass privilege-gain
- Basis — asserted — **Downgraded from `verified`: the "meta not settable from source" claim is a parser assumption, not run.**

**`io_lib:format(...)` in `format_error/1` and warning payloads (lines 361–373) — diagnostic formatting**
- Reason — Format strings are compile-time constants; arguments are atoms (`Name`, `Receiver`), integers (`Arity`), and a deprecation `Message`, rendered with non-recursive directives (`~ts`/`~s`/`~B`) — readable in-file, so no format-string injection. A `Message` from a dependency's `@deprecated` attribute could carry terminal escapes into compile-warning output, but controlling that attribute requires controlling a compiled dependency (compile-time code execution).
- Axes examined: format-directive injection via `~s`/`~ts` (in-file: constant format strings) | Not examined: terminal-escape / log-injection via `Message` (dismissed on threat-model policy — provenance is a compiled dependency — not executed; I did not render an escape-bearing `Message` through the warning path to observe output)
- Basis — asserted — **Downgraded from `verified`: the injection-safety of the constant format strings is in-file, but the `Message`-provenance dismissal is policy, not run.**

**`prune_stacktrace/4`, `find_imports_by_name/*`, `find_import_by_name_arity/*` — recursion / lookup (lines 290–357)**
- Reason — `prune_stacktrace` recurses over a VM-truncated runtime stacktrace (bounded); the import walkers recurse over the environment's import ordsets, whose size is bounded by the number of `import` statements. The recursion structure and termination clauses are fully readable in this file; the functions only build tuples / raise controlled `file_error`s — no dangerous sink.
- Axes examined: recursion termination / no dangerous sink (in-file control flow, read in full) | Not examined: that ordset size is truly source-bounded (leans on `elixir_import` population — read, not executed), quadratic accumulation on adversarial ordsets (not executed)
- Basis — asserted — **Downgraded from `verified`: termination is in-file, but the input-size bound leans on a neighbour and was not exercised.**

### Did I verify this?

One axis is verified by execution in this session: atom-table growth for `macro_name` (re-deriving an interned name → delta 0 over 100 000 calls; 5 000 distinct novel names → delta 5 000), which establishes that the `list_to_atom` sink grows the table only with distinct novel inputs. I cite the commands I ran, not a prior pass's claim.

Everything else is `asserted`. The dismissals rest on (1) this file's own control flow, which I read in full, and (2) folder-threat-model policy (compiling untrusted source grants compile-time execution; loading/introspecting source-referenced modules and invoking their macros is contract, not escalation). I re-read the reached callees (`elixir_def:fun_for/6`, `elixir_utils:macro_name/1` and `caller/4`) to confirm the guards *exist and have the shape* the dismissals assume — but per this pass's rule, reading a guard does not clear the axis of *which adversarial inputs* would make it disagree with the caller. Those axes remain `Not examined`, and I did not have a tractable way to execute them (they require driving crafted source/AST through the full tokenizer→parser→expander→def pipeline). Because none of the reads or the one experiment produced contrary evidence, no dismissal is promoted to a finding.

### Dropped in re-review

None. Every ruled-out entry from the prior report is preserved. The prior report had no findings to preserve or drop, and this pass adds none.

### Upgraded from hedged assumption in first pass

None upgraded to findings. Per the pass's downgrade-preference rule, promoting a dismissal to a finding requires executed contrary evidence, and none exists. Instead, the hedge is resolved in the *safe* direction: the intervening pass's over-claimed `verified` / `Not examined: none` labels — which were earned by reading neighbours — are corrected to `asserted` with the untried axes restored to `Not examined`, so a later pass can re-examine them. The single genuinely executable axis (atom-table growth) was re-run and is cited as `verified`.

## Risks / caveats

- This review holds under the folder threat model's boundary: a party who supplies untrusted Elixir *source* (or an untrusted quoted AST to `eval_quoted`/`expand`) obtains compile-time code execution, so macro invocation here is not an escalation. If a consumer wrongly treats `Code.string_to_quoted` + expansion as a sandbox, every macro-dispatch path in this file runs attacker code — a documented caller contract, not a defect in `elixir_dispatch.erl`.
- The `check_deprecated`/`is_macro` paths depend on the referenced module's `__info__/1`; both call sites `catch error:_` (lines 380-384, 397-401, read directly), so a malformed `__info__` degrades to "no deprecation / not a macro" rather than crashing.
- Honest scope of this pass: I read this file in full and re-read the exact reached callee functions in `elixir_def`/`elixir_utils`; I executed exactly one differential (atom-table growth). The `Not examined` axes on the entries above are the inputs a read cannot cover and that I could not practically execute in isolation — they are recorded as untried, not as cleared.
