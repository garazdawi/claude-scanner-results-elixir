# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/of.ex
<!-- sha256: 372dffe737c24b0f87ebfa1a354891c7db290b86398d952a1a905ae195169beb -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:06:23Z_

Everything is confirmed. Key facts established:

- **Verified exponential blowup** localized to this file's `closed_map/5` → `cartesian_map/1` path: 15 union-typed variables cost 0.18s as literal-keyed values/tuple elements but **17.85s when used as map keys**; N=18 did not finish in ~100s.
- **Trust boundary**: `Module.Types:infer` runs from `elixir_module.erl:205` on already-expanded definitions during module compilation; there is no `Code` API that type-checks without compiling, and compiling executes the module body (proven). So reaching this requires `mix compile` of the source.
- The unguarded `struct.__info__` in `impl/2` receives only protocol-impl target modules (built-in type modules / user Elixir structs) and won't realistically raise — ruled out.

Here is the report.

---

# Security Review — `lib/elixir/lib/module/types/of.ex`

## Summary
The set-theoretic type-checker's closed-map builder expands map keys with finite-atom-union types into a full Cartesian product, giving a tiny source file exponential compile-time cost; no other security-relevant sink (atom creation, code/command execution, path/archive handling, deserialization) is present in this file.

## Security

#### `CSCAN-of-cartesian-map-exponential`
- **Finding ID:** `CSCAN-of-cartesian-map-exponential`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `lib/elixir/lib/module/types/of.ex:378-388` (`closed_map/5`) and `:460-468` (`cartesian_map/1`); accumulation site `:371`.
- **Class:** Resource consumption (algorithmic complexity / compile-time DoS, CWE-407 / CWE-1333-adjacent)
- **Issue:** In `closed_map/5`, every map pair whose key type is a *finite union of atoms* is pushed onto the `multiple` list (line 371). The final map is then built by enumerating `cartesian_map(tail)` (lines 383-388, 460-468), i.e. the full Cartesian product of all multiple-key entries, producing `∏ |keys_i|` distinct `closed_map/1` terms which are then folded with `opt_union/2`. With N keys each carrying k atom possibilities, this is `kᴺ` generated maps — exponential in the number of such keys. The code comment at lines 345-348 acknowledges the imprecision ("those should be very uncommon") but imposes **no bound** on the expansion.
- **Impact / threat:** A ~20-line module (a plain map literal whose keys are variables typed as small atom unions, e.g. `k = if x, do: :a, else: :b`) drives the compiler into exponential CPU and memory during type inference. This runs during `mix compile` (and therefore during `mix deps.compile`, CI that compiles branches/PRs, and any editor/LSP that compiles the project), hanging the build. The trigger is an innocuous-looking map literal with no obviously malicious construct, so it reads as an accidental compiler bug rather than a payload — a stealthy build/CI denial-of-service reachable through any compiled dependency or source file. **Caveat (see Risks):** reaching the type checker requires compiling the source, and compilation already executes the module body, so this DoS does not cross a boundary that plain compilation doesn't already cross; it is a stealthier, lower-privilege sibling of the code execution an attacker who controls compiled source already has.
- **Evidence:** `verified`.
  - Escalation (all other conditions equal, only key-count N varies):
    ```
    $ TIMEFORMAT='%R sec'; time elixirc blow_$N.ex   # %{k1=>1, ...} with k_i = if x, do: :a, else: :b
    N=10 : 0.207 sec
    N=15 : 18.625 sec
    N=18 : (did not finish within ~100s; run timed out)
    ```
  - Isolation to this file's path (same 15 union-typed vars):
    ```
    ctrlA (N=15, vars as *values*, literal atom keys):  0.181 sec
    ctrlB (N=15, vars in a tuple, no map):               0.164 sec
    blow_15 (N=15, vars used AS map keys):              17.850 sec
    ```
    Only the union-typed-**keys** case is slow, confirming the cost is the `multiple` → `cartesian_map` → `closed_map` expansion in `of.ex`, not the surrounding `if`/inference.
  - Reachability/boundary: `grep` shows `'Elixir.Module.Types':infer(...)` invoked from `elixir_module.erl:205` on already-expanded `AllDefinitions`; no `Code` API type-checks without compiling; and `elixirc` of a module prints a compile-time `IO.puts`, proving the body executes when the checker runs.
- **Suggested fix:** Bound the expansion in `closed_map/5`. Before the comprehension at line 383, compute the product `Enum.reduce(multiple, 1, fn {keys, _}, acc -> acc * length(keys) end)`; when it exceeds a fixed threshold, do not enumerate — instead widen the `multiple` entries into a single domain-keyed entry (merge their keys into `domain` with the `opt_union` of their value types), yielding an open/domain map that is a sound supertype. This matches the module's own note that the precise distribution "should be very uncommon" while capping cost at a constant.

### Ruled out
- **Unguarded `struct.__info__(:struct)` in `impl/2` (line 294)** — reachable via `impl/1` at `apply.ex:2276`, but `mods` there come from `mod.__protocol__(:impls)` (`{:consolidated, mods}`): protocol-implementation targets, i.e. the built-in type modules already matched by the `impls` clauses (Atom, List, Map, …) or user-defined Elixir structs, all of which define `__info__/1`. `__info__(:struct)` returns `nil` (does not raise) for a loaded non-struct Elixir module, and the path only runs while formatting a protocol-mismatch error message. No realistic input reaches an `UndefinedFunctionError`. Basis: read of the single call site and its argument domain.
- **`Code.ensure_loaded?/1` + dynamic dispatch (`struct.__info__`, `struct.__struct__` via `Module.Types.Apply.signature`) in `impl/2` and `struct_info/6`** — loads modules and calls compiler-generated introspection during type-checking. The modules are names drawn from the program under compilation; loading them (and any `@on_load`) is inherent, expected compiler behavior for type-checking `%Struct{}`/remote calls, not an externally-injected callable. No atomization of external bytes occurs (`atom([struct])` wraps an already-existing atom). Basis: read of both functions and the `Descr.atom/1` constructor.
- **Atom creation / `String.to_atom` / `Module.concat`** — none present; grep for `String.to_atom|binary_to_atom|list_to_atom|Module.concat|Code.eval|Code.compile|apply(` returns no matches in the file. Basis: grep + full read.
- **`format_diagnostic/1` string interpolation (lines 747-895)** — interpolates type/atom values into human-readable compiler diagnostics via `IO.iodata_to_binary`; the result is a warning/error message, not fed to any downstream interpreter (shell, query, EEx, regex). Not a template-injection sink. Basis: read; output sink is the diagnostics channel only.
- **Type predicates (`compatible?`, `empty?`, `gradual?`, `map_fetch_key`)** — a wrong answer is a type-soundness/precision bug, not a security "is-this-safe" gate on untrusted data in the CWE sense. Out of scope for this security review. Basis: read.

## Risks / caveats
- **Severity hinge — the compile-time boundary.** The one finding is a genuine, verified exponential-time/space algorithmic-complexity bug, but it lives in a phase (`Module.Types:infer`) that runs only *after* macro expansion during `defmodule` compilation. I verified there is no public API that type-checks quoted/expanded AST without compiling, and that compiling a module executes its body. Consequently, an attacker who can reach this sink can already run arbitrary code at compile time; the DoS is strictly weaker than that capability. I rated it **Medium** because the review's own severity guidance up-weights sinks reachable during `mix compile`/`mix deps.compile` (weaker expectations, wider reach: dependencies, CI compiling untrusted branches, LSPs), and because the trigger is a benign-looking map literal. A reviewer who weighs the "compilation ⇒ RCE" redundancy more heavily could justifiably downgrade this to Low; I have stated the caveat explicitly rather than hiding it.
- **Base rate of the expansion.** Triggering the exponential requires map keys whose inferred types are *finite atom unions* with several possibilities each; ordinary code (literal `key:` atoms, string/integer keys, single-atom keys) takes the fast single-key path, as the control cases show. This is why it has evidently gone unnoticed — it is a crafted-input property, which is exactly the threat model in scope here.
- **k vs N.** I escalated the key count N at k=2 and observed timeout by N≈18. I did not separately sweep k (atoms per key); the closed-form `∏|keys_i|` makes larger k strictly worse, so this does not affect the conclusion, but the exact crossover for higher k is un-measured.
- **Fix threshold.** The suggested widening bound is sound (an open/domain map is a supertype), but the precise threshold and its interaction with downstream precision (e.g. `map_fetch` on the widened result) should be chosen by the maintainers; I did not implement or test the fix.

