# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module.ex
<!-- sha256: 9f30a3ab43608e96e71383a5eeb01a4df3d4212f5c5c2aeaa8c88aeab3f7b486 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module.ex
<!-- sha256: 9f30a3ab43608e96e71383a5eeb01a4df3d4212f5c5c2aeaa8c88aeab3f7b486 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T (hedge-resolution pass)_

## Summary
`Module` is a compile-time metaprogramming API whose only documented, runtime-reachable, binary→atom sink is `Module.concat/1,2` (CWE-770), now confirmed end-to-end against the delegated `:elixir_aliases` implementation. A second, secondary atom-creation family (`String.to_unsafe_atom` in signature building) is reachable only through `@doc false`/deprecated internal entry points; the first pass hedged it out of scope, and this pass upgrades it to a low-confidence finding rather than leave the assumption unverified. No OS/file/eval-of-external-data sinks exist in the file.

## Security

#### `CSCAN-module-concat-atom-exhaustion`
- **Finding ID:** `CSCAN-module-concat-atom-exhaustion`
- **Severity:** Low
- **Confidence:** high
- **Location:** `module.ex:979` (`concat/1`) and `module.ex:1008–1011` (`concat/2`), delegating to `:elixir_aliases.concat/1` (`elixir_aliases.erl:175`)
- **Class:** Atom creation (CWE-770)
- **Issue:** `Module.concat/1,2` unconditionally interns a **new atom** for the resulting alias, including when the components are attacker-supplied binaries. The delegation target is now verified, not assumed: `elixir_aliases.erl:175` is `concat(Args) -> binary_to_atom(do_concat(Args), utf8)` — an unconditional `binary_to_atom` with no existence check. The checked counterpart `safe_concat/1` is `elixir_aliases.erl:176`: `binary_to_existing_atom(do_concat(Args), utf8)`, which interns nothing new. `do_concat` (`elixir_aliases.erl:178–198`) only string-concatenates with `<<Acc/binary, $., ...>>`; the sole atom-minting step is the final `binary_to_atom`.
- **Impact / threat:** An attacker who can steer the binary arguments (crossing the network/file→VM boundary in a calling application) can force one new permanent atom per distinct input. At the default atom limit (`1_048_576`) this crashes the entire BEAM node — full-node DoS, not a single-process crash. This is the documented, by-design behavior of `concat`; the defect materializes in a caller that forwards untrusted data. Reported per the standing "always flag `Module.concat`" rule.
- **Evidence:** `verified` (runtime + delegated-source read).
  - Runtime (prior pass, re-affirmed): `Module.safe_concat(Foo, "Zzq1410Uniq")` raised `"1st argument: not an already existing atom"`; `Module.concat(Foo, "Zzq1410Uniq")` returned `Foo.Zzq1410Uniq` (`is_atom == true`).
  - Growth: 50k distinct external strings → 50186 atoms; `atom_limit: 1048576`.
  - Delegated source: `elixir_aliases.erl:175–198` read this pass — confirms `binary_to_atom` (concat) vs `binary_to_existing_atom` (safe_concat). This resolves the first-pass caveat "I did not audit `:elixir_aliases`… assumed correct."
- **Suggested fix:** No change to module.ex logic required (behavior correct and documented). Harden callers: (a) add an atom-exhaustion warning to the `concat/1,2` docs pointing at `safe_concat/1,2`; (b) callers should use `safe_concat/1,2` or an allow-list whenever a component is externally derived.

#### `CSCAN-module-signature-unsafe-atom` — **Upgraded from hedged assumption in first pass**
- **Finding ID:** `CSCAN-module-signature-unsafe-atom`
- **Severity:** Low
- **Confidence:** low
- **Location:** `module.ex:1151`, `1162`, `1195`, `1229` (`simplify_var`, `simplify_module_name`, `expand_key`, `merge_signature`), each calling `String.to_unsafe_atom/1`
- **Class:** Atom creation (CWE-770)
- **Issue:** These four sites mint atoms via `String.to_unsafe_atom` (the unchecked `binary_to_atom` variant) while building the parameter signature stored in docs. The first pass dismissed them as out of documented-API scope; per the re-review rule preferring an upgrade over an unverified dismissal, they are recorded here as a low-confidence finding. Reachability was confirmed this pass: the only in-file callers of `build_signature` (`module.ex:1064`) are `compile_definition_attributes/6` (`module.ex:1854`, `@doc false`, commented "private and must be used only internally") via `compile_doc` (`module.ex:1882`), and `add_doc/6` (`module.ex:1841`, `@doc false` + `@deprecated "Use @doc instead"`) via `compile_doc` (`module.ex:1846`).
- **Impact / threat:** Bounded and low. Inputs are compiler-derived: `simplify_var` operates on a def's variable names, `simplify_module_name` on a split/underscored module name, and `expand_key`/`merge_signature` on `"#{key}#{count}"` / `"arg#{i}"` where the counters are bounded by argument count (arity ≤ 255). A module reaching this path is already executing its own macros (full build-time code execution), so it is not a privilege/scope escalation. The residual concern is only that these are unchecked `binary_to_atom` sinks on the compile path; harm requires the (deprecated/internal) entry points to be driven with adversarial, non-arity-bounded names, which the documented pipeline does not do.
- **Evidence:** `verified` for reachability and sink identity (reads of `module.ex:1064,1146–1200,1839–1882`); `asserted` for exploitability (no executed input that produces unbounded distinct atoms here — the counter/arity bounding was read, not fuzzed).
- **Suggested fix:** None required for shipped behavior. If the internal doc-compilation entry points were ever exposed to non-source-derived signatures, switch these four sites to a checked/interned-name strategy.

### Ruled out
- **`eval_quoted/4` (line 869) and `create/3` (line 937)** — code-execution/compile sinks documented to evaluate/compile *caller-supplied quoted expressions*. Harm requires the victim to pass attacker-controlled AST — the eval-on-your-own-input carve-out (caller bug, not a module.ex defect). Both gate on `assert_not_compiled!`/`env_for_eval`. The first-pass note that I "did not audit `:elixir.eval_quoted`/`:elixir_module`" is retained as a trust boundary *by contract, not by assumption*: these functions are documented to run code, so reading their internals cannot change the verdict — the dismissal rests on the documented execute-caller-code contract, not on an unverified belief about their implementation. Basis: read + documented contract.
- **`String.to_unsafe_atom` in signature building (lines 1151, 1162, 1195, 1229)** — no longer ruled out; **promoted** to `CSCAN-module-signature-unsafe-atom` above (low confidence).
- **`split/1` (line 1821)** — accepts a binary but produces only strings via `String.split`; verified it creates **0** atoms, raises `ArgumentError` on non-`Elixir.`-prefixed input (`module.ex:1831–1837`), and returns `[""]` for a trailing dot. No atom/injection sink. Basis: `verified`.
- **`safe_concat/1,2` (lines 1033, 1057)** — the safe predicate; delegates to `:elixir_aliases.safe_concat` = `binary_to_existing_atom(...)` (`elixir_aliases.erl:176`), which interns nothing new (also verified at runtime: raises `"not an already existing atom"` for unseen names). Basis: `verified` (runtime + delegated-source read this pass).
- **`preprocess_attribute(:behaviour, …)` → `Code.ensure_compiled/1` (line 2255); `check_module_for_overridable` → `Code.ensure_loaded?/1` (line 1553)** — operate on an already-existing module **atom** (guarded `is_atom`), during compilation of an open module only (`__put_attribute__` asserts not-compiled). No binary→atom conversion, no untrusted-string→module path. Basis: read.
- **ETS attribute writes (`register_attribute/3` line 1784, `put_attribute` family lines 2134–2227) — check-then-act (`insert_new`/`update_element`)** — the data tables belong to the single module currently being compiled and are manipulated from its own compile process; the folder threat model establishes compilation is per-caller-process, so there is no documented multi-writer/untrusted-input path and the TOCTOU is not attacker-reachable. Basis: read + folder threat model.
- **`concat/1` accepting a non-`binary|atom` list, `put_attribute` value shapes, etc.** — reaching a crash requires arguments that violate the documented `@spec`/guards, which the rules exclude. Basis: read + spec.

## Risks / caveats
- The atom-exhaustion consequence of `concat` is confirmed against the built toolchain and now against the delegated `:elixir_aliases` source (`binary_to_atom`). The *severity to a given deployment* still depends on whether a calling application forwards untrusted data into `Module.concat` — this is the intrinsic property of a CWE-770 sink, not an unverified assumption: the sink is confirmed here; only the presence of an untrusted *caller* lives in other files. Rated Low as-shipped (documented behavior, safe alternative provided), with a full-node DoS as the misuse consequence.
- Delegated modules not read this pass: `:elixir_module`, `:elixir_def`, `:elixir_overridable`, `Kernel.Typespec`, `Module.Behaviour`. None sit on a binary→atom or OS/file/eval-of-external-data path reachable from this file's documented API with untrusted input; the sinks that do (`concat`/`safe_concat` → `:elixir_aliases`) were read and verified. These remain trust boundaries by scope, and no hedged *finding* depends on them.

## Did I verify this
- **`concat`/`safe_concat` atom behavior:** Verified — runtime (atom minted vs refused; 50k→50186 growth) **and** delegated source `elixir_aliases.erl:175–198` (`binary_to_atom` vs `binary_to_existing_atom`).
- **Signature-building sink identity + reachability:** Verified by read — `String.to_unsafe_atom` at `module.ex:1151/1162/1195/1229`; callers `build_signature` ← `compile_doc` ← `compile_definition_attributes` (`@doc false`) / `add_doc` (`@doc false` + `@deprecated`). Exploitability (unbounded distinct atoms) **not** executed — hence low confidence, not a dismissal.
- **`split/1`:** Verified — 0 atoms, `ArgumentError` path read at `module.ex:1831–1837`.
- **`eval_quoted`/`create`:** Not executed; dismissal rests on the documented execute-caller-code contract, which read confirms.

## Dropped in re-review
- No prior findings dropped. The prior report's single finding (`CSCAN-module-concat-atom-exhaustion`) is retained and strengthened. The prior ruled-out entry for the `String.to_unsafe_atom` signature-building paths was **not dropped but upgraded** to a low-confidence finding (`CSCAN-module-signature-unsafe-atom`), resolving the first-pass hedge "if that scope were widened … would warrant a second look."
