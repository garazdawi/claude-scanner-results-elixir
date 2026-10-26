# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types.ex
<!-- sha256: 0b46de6955ab6da422d482881ab5052d8c391f3838d83e9531a727addf01d3e4 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types.ex
<!-- sha256: 0b46de6955ab6da422d482881ab5052d8c391f3838d83e9531a727addf01d3e4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`Module.Types` is a compiler-internal type-inference/checking pass over already-structured function definitions. The first pass concluded "No findings" and rested that entirely on a trust-boundary claim: that both entry points (`infer/7`, `warnings/6`) only ever receive the in-memory, freshly-macro-expanded definitions of the module being compiled, so any resource-consumption or dynamic-dispatch sink is "not a distinct boundary crossing."

Re-reading the two documented callers **partially falsifies that claim**. `infer/7` (called from `elixir_module.erl:205`) does indeed only run on live compilation state. But `warnings/6` has a *second* caller path — `Module.Types.ParallelChecker.verify/2` (a `@spec`-documented public function) → `spawn_and_register_cache/6` → it reads a `.beam` file from disk (`File.read`, `parallel_checker.ex:118`), pulls `debug_info` out of it (`parallel_checker.ex:119-124`), and feeds the resulting `definitions`/`attrs` into `Module.Types.warnings/6` (`parallel_checker.ex:282`). On that path the definitions were macro-expanded in *some earlier, possibly foreign* build, and reading them back does **not** re-run macros in the checking process. The first pass explicitly flagged this exact scenario ("if some future caller fed this module definitions parsed from an untrusted `.beam`… the resource-consumption and dynamic-dispatch items would need re-evaluation") and then dismissed it as hypothetical. It is not hypothetical; the caller exists today. I therefore upgrade it to a low-confidence finding.

## Security

### Finding 1 — `warnings/6` processes definitions/attributes read from an on-disk `.beam` without re-running macros (resource consumption + attacker-named module load)

**Upgraded from hedged assumption in first pass.** (Sink class: resource consumption / DoS, secondarily dynamic code-load.)

**Confidence: Low.**

- **Path:** `ParallelChecker.verify/2` (`parallel_checker.ex:190`, documented `@spec`, argument `runtime_files :: [{module(), Path.t()}]`) → `spawn_and_register_cache/6` (`:101`) → `File.read(location)` (`:118`) → `:beam_lib.chunks(binary, [:debug_info, ~c"ExCk"])` (`:119`) → `backend.debug_info(:elixir_v1, module, data, [])` (`:121`) → `module_map_to_module_tuple/1` (`:124`, `:299`) → `spawn_checker` → `check_module/4` (`:266`) → `Module.Types.warnings(module, file, attrs, definitions, no_warn_undefined, cache)` (`:282`). Confirmed by reading each hop; `definitions` and `attrs` in the tuple come solely from the `.beam`'s `debug_info` and attribute chunks, not from live compilation.
- **Consequence A (DoS):** `group_clauses_by_return`/`group_clause_by_return`/`union_args` (types.ex:571-611) and `add_inferred`/`group_clauses` (types.ex:443-559) are O(n²) in the clause count of a single function. A hand-crafted `.beam` whose `debug_info` encodes a function with a very large number of clauses would make `warnings/6` burn quadratic CPU during type checking — without the attacker's code ever executing in the victim process (unlike the compile path, where macro expansion already ran attacker code).
- **Consequence B (module load):** `impl_for/1` (types.ex:136-148) reads the `@__impl__` attribute (which on this path comes from the `.beam`) and, if present, calls `Code.ensure_loaded?(protocol)` (types.ex:139). A crafted `.beam` can name an arbitrary `protocol` module, causing the checker to load it (and run its `@on_load`, if any) merely by type-checking the artifact. `attributes` are narrowed to `[:__protocol__, :__impl__]` at `parallel_checker.ex:324`, which preserves exactly this attacker-influenceable key.
- **Why Low, not higher:** to reach `verify/2` with a hostile `.beam`, an attacker must already have placed that `.beam` on the victim's code path. In the normal workflow that same `.beam` would also be *loaded and executed* as a dependency, which is strictly greater capability than type-checking it; the O(n²) work and the `ensure_loaded?` are marginal amplifications over "you already run this module's code." I could not establish a realistic deployment in which an untrusted `.beam` is type-checked via `verify/2` but never loaded/run. Hence low confidence rather than a firm DoS finding. The file itself has no per-clause guard/bound, so the mitigation (if any) would live in the caller or in a size cap that I did not find.

### Remaining sink inventory (unchanged from first pass, still holds)

- **Atom creation** — none. No `String.to_atom`/`binary_to_atom`/`Module.concat`; all `fun_arity` names are pre-existing atoms from the definition tuples.
- **Command exec / file ops (in this file) / archive / deserialisation / network / crypto / Unicode** — none present in `types.ex`. (Note: the `File.read` + `binary_to_term` that feed Finding 1 live in `parallel_checker.ex`, not here.)
- **Template / interpolation** — `internal_error!/9` (types.ex:620), `format_error/1`, `format_diagnostic/1` build human-readable diagnostics via `#{}`; none reach a downstream interpreter. See *Ruled out*.
- **Shared mutable state** — reads `:elixir_config.get(:infer_signatures)` (types.ex:42); no writes to global/ETS state from this file.

## Ruled out

- **`internal_error!/9` embedding source via `Macro.to_string/1` (types.ex:620-637)** — emits the code under check into a "please report this bug" `RuntimeError`. The audience is the developer compiling/checking their own (or a dependency's) code; no cross-tenant or secret data is in scope.
  - Axes examined: content-source (the def AST being formatted), audience (same developer/build process that invoked the checker).
  - Not examined: whether, on the `verify/2` `.beam` path, `Macro.to_string/1` on adversarially-shaped AST from a hostile `.beam` can itself be driven into pathological cost or a crash loop distinct from Finding 1's O(n²) (I did not push a crafted AST through `Macro.to_string/1` and measure).
  - Status: **asserted.**

- **`Keyword.fetch!(meta, :super)` / `format_error`/`format_diagnostic` pattern-match raises (types.ex:353, 722-759)** — a malformed `defs`/`meta` shape raises; on the infer/warnings paths this is caught and rethrown by `internal_error!` as a reportable bug rather than silent corruption.
  - Axes examined: shape-contract origin (compiler builds the tuples on the compile path).
  - Not examined: whether a hostile `.beam` on the `verify/2` path can supply a `meta`/`defs` shape that raises *outside* the `try/rescue` in `default_local_handler`/`infer_local_handler` (e.g. the `Keyword.fetch!(meta, :super)` at types.ex:353 is inside the `try` at :347, but `warn_unused_clauses`/`List.keyfind`/`Enum.fetch!` at types.ex:245-270 are not wrapped) and thereby crash the checker process for that module. I did not construct such a `.beam` and run it to see whether the crash is contained to the per-module checker process (it is spawned, `parallel_checker.ex:83`) or propagates. If it is contained, this is a non-finding; if it can take down the checker, it is a sibling of Finding 1.
  - Status: **asserted.**

- **`impl_for/1` dynamic dispatch, on the `infer/7` compile path only (types.ex:136-148)** — when reached from `elixir_module.erl:205`, `attrs` is the live in-memory attribute list of the module being compiled, whose macros have already executed arbitrary build-time code; loading a referenced protocol there is not a distinct boundary crossing. (The `.beam`-sourced variant of this same code is *not* ruled out — it is Consequence B of Finding 1.)
  - Axes examined: caller (`elixir_module.erl:205`), input origin on that caller (live `Attributes`).
  - Not examined: none for the compile path — but note the *other* caller (`warnings/6` via `verify/2`) is handled as Finding 1, so this ruling is scoped strictly to `infer/7`.
  - Status: **verified (compile path only).**

## Did I verify this

- **Verified by reading the callgraph neighbours** (as instructed): `elixir_module.erl:180-239` confirms `infer/7` receives `AllDefinitions`/`Attributes`/`E` from live compilation. `parallel_checker.ex:82-143, 266-328` confirms the *second* `warnings/6` caller reads definitions/attributes from an on-disk `.beam` via `File.read` + `:beam_lib.chunks` + `backend.debug_info`. This read is what falsified the first pass's "boundary is verified, not assumed" and forced the upgrade.
- **No differential-execution axis was tried.** The residual axes on the ruled-out entries (can a crafted `.beam` drive `Macro.to_string/1` pathologically; can it raise outside the `try/rescue` and crash/contain the spawned checker) are single-implementation reachability/robustness questions, not two-implementation disagreements. There is no pair of implementations to push a value through and diff, so I did not run the Erlang toolchain for them; they remain honestly `asserted` with their `Not examined` lists populated. I did not fabricate a "verified" status for any entry whose axes I did not exercise.
- **Finding 1's reachability was established structurally (by reading), not by executing an end-to-end exploit.** I did not construct a malicious `.beam` and measure the O(n²) blow-up or the `ensure_loaded?` load, and I could not identify a real deployment that type-checks an untrusted `.beam` without also running it — which is precisely why it is rated **Low** rather than a confirmed DoS.

## Dropped in re-review

- **Dropped: the unqualified "No findings" / "trust boundary fully closed" conclusion.** New evidence: `parallel_checker.ex:118-124, 190, 282` shows `warnings/6` definitions can originate from an on-disk `.beam` decoded via `debug_info`, with no macro expansion in the checking process. The first pass named this scenario and dismissed it as a hypothetical future caller; the caller exists now, so the dismissal is replaced by Finding 1 rather than dropped silently. No other prior finding is dropped (the prior report had none).

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 2 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`Keyword.fetch!(meta, :super)` / `format_error` pattern-match raises** — a malformed `defs`/`meta` shape could raise, but that violates the caller's (compiler's) internal contract and is caught by `internal_error!`; it is not reachable via any documented public API with attacker-shaped input.

- **O(n²) clause grouping (`group_clause_by_return`/`union_args`/`add_inferred`, types.ex:552-611)** — quadratic in the clause count of a single function, which requires proportionally large source. It runs at compile time on code that is being compiled (and thus executed via its macros) regardless. A dependency wishing to burn build CPU can do so directly in a macro body far more cheaply; this pass adds no amplified DoS primitive over an untrusted-input boundary. Basis: algorithmic inspection plus the compile-time-only reachability confirmed above.
