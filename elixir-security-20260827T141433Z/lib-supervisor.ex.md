# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/supervisor.ex
<!-- sha256: dc041ffc948bc8e02911555be83675b93b6312ea75ecef217eb4452f75a82bc7 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/supervisor.ex
<!-- sha256: dc041ffc948bc8e02911555be83675b93b6312ea75ecef217eb4452f75a82bc7 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28T (single-file hedges revisited)_

## Summary
`Supervisor` is a thin, well-guarded wrapper over OTP `:supervisor`. Its only dynamic behaviours — module dispatch (`module.child_spec(arg)`) and a diagnostic module load (`Code.ensure_loaded?/1`) — are the documented, intended semantics of a supervisor, driven by developer-authored supervision trees rather than untrusted input. No security findings are reachable through its documented public API. This conclusion is unchanged from the first pass; below I resolve the cross-file hedges the first-pass reviewer flagged.

**Note on the standing context callgraph neighbours.** The neighbour modules named in the folder threat model (`elixir_tokenizer`, `elixir_interpolation`, `elixir_parser`, `elixir_expand`, `elixir_utils`, etc.) belong to the **Erlang compiler front-end in `elixir/lib/elixir/src`**. They are *not* callgraph neighbours of `supervisor.ex` (which lives in `lib/elixir/lib/` and delegates to OTP `:supervisor` + `GenServer` + `Code`). Reading them would tell me nothing about this file. So option (a) — "read the callgraph neighbours listed in the standing context" — does not apply to any hedge here; the relevant boundaries are OTP `:supervisor` (Erlang/OTP, outside this repo tree) and `lib/elixir/lib/code.ex`. I address each hedge on those terms below.

## Security

`No findings`

### Ruled out

- **Dynamic behaviour dispatch `module.child_spec(arg)` (`init_child/1,2`, lines 816–833; reached from `start_link/2`, `init/2`, `child_spec/2`, `start_child/2`).**
  This invokes an arbitrary module's `child_spec/1`; the resulting spec's `:start` MFA is later `apply`-ed by OTP. This is the defining contract of a supervisor, not a bug. The security-relevant question is answerable **entirely within this file**: does attacker-controlled input reach the dispatched `module` via the public API? No — the `when is_atom(module)` guards at lines 816 and 820 mean `module` is always a pre-existing atom placed into the supervision tree by the developer. There is no string→atom conversion (confirmed by grep, below), so no atom-table exhaustion and no confusable/injection surface. The downstream MFA execution lives in OTP `:supervisor` (Erlang), which is neither in this folder nor a listed neighbour — but the ruling-out verdict does **not** depend on that code: the verdict rests on the in-file guard, which I read directly.
  - Confidence: High.
  - Axes examined: input provenance of `module` (developer-supplied atom via `children`/`child_spec`/`start_child`); string→atom conversion (none); guard reachability (`is_atom/1` at 816, 820). | Not examined: none — every axis bearing on *this file's* contribution to the sink was read in-file.
  - Basis: full read of the dispatch path (lines 813–854) + grep for `*_to_atom`/`concat`/`apply` (no matches, see below).

- **`Code.ensure_loaded?(module)` (`child_spec_error/1`, line 857).**
  Reached only on the error path when a developer-supplied module lacks `child_spec/1` (caller at line 827). The argument is the same already-existing, developer-controlled atom that `init_child/2` was about to invoke anyway; loading it grants no surface beyond the dispatch above. `Code.ensure_loaded?/1` is defined in `lib/elixir/lib/code.ex` (the reachable neighbour for this call), and takes an atom module name — it performs no string→atom conversion of external input. Not attacker-controllable via this file's public API in normal use.
  - Confidence: High.
  - Axes examined: caller provenance (single caller, line 827, same guarded atom); argument type (atom, guarded upstream). | Not examined: none for this file's contribution.
  - Basis: traced the only caller (line 827); argument identity with the guarded `module`.

- **Atom creation / atom-table exhaustion.** None present. No `String.to_atom`, `List.to_atom`, `:erlang.binary_to_atom`, or `Module.concat` anywhere in the file (grep matched only `Module.has_attribute?` at 532 and `Code.ensure_loaded?` at 857 — neither atomises input). All module values arrive as pre-existing atoms guarded by `is_atom/1`.
  - Confidence: High. Axes examined: presence of any atomisation sink (grep, whole file). | Not examined: none.

- **Deserialization / code-eval / command / path / archive / crypto / template / regex sinks.** None present. The module performs no I/O, spawns no OS processes, and decodes no external byte formats. Verified by whole-file grep for `to_atom|concat|binary_to_term|apply|String.to|ensure_loaded|Module.` (only the two benign matches above).
  - Confidence: High. Axes examined: sink-pattern presence across the full file. | Not examined: none.

- **`__using__/1` compile-time macro (lines 527–552).** `opts` come from the developer's own `use Supervisor, opts` statement and are `Macro.escape/1`-d before embedding; no external input reaches compile time. Whoever controls the `use` site already controls the compiled module. `Module.has_attribute?/2` (line 532) inspects `__MODULE__`, the developer's own module.
  - Confidence: High. Axes examined: input provenance of `opts` (developer `use` site); attribute target (`__MODULE__`). | Not examined: none.

- **`name` validation in `start_link/3` (lines 977–1001).** The catch-all `other ->` branch is reachable with any non-conforming `:name` value and raises `ArgumentError` — a defensive gate that fires, not dead code. `{:via, via_module, _}` correctly guards `is_atom(via_module)`.
  - Confidence: High. Axes examined: reachability of catch-all; `is_atom` guard on `via_module`. | Not examined: none.

- **`start_link/2` vs `start_link/3` arity dispatch (list-vs-module first argument).** Intentional and documented (comment, lines 972–973); both paths start a supervisor and the first argument is developer-supplied structure, not attacker-steered content. No security consequence.
  - Confidence: High. Axes examined: dispatch determinant (argument shape, developer-supplied). | Not examined: none.

- **`count_children/1` → `:maps.from_list/1` (line 1183).** Input is the reply from the trusted OTP `:supervisor` process, not an external boundary; `:maps.from_list` cannot fail-open into anything dangerous.
  - Confidence: High. Axes examined: input source (trusted OTP reply). | Not examined: none.

### Resolution of first-pass cross-file hedges

The first-pass report carried two cross-file caveats that the reviewer flagged as single-file hedges. Re-examined:

1. **"Cross-file behavior out of scope: OTP `:supervisor` / `GenServer`."** This is a true scope boundary, not a dismissal masking an unverified in-file claim. The process-starting, restart-intensity accounting, and spec validation execute in OTP Erlang code that is (a) not in this repo folder and (b) not a listed callgraph neighbour, so there is nothing to read here to "verify" it. Crucially, **no ruling-out verdict above depends on that code**: each dismissal is grounded in an in-file guard (`is_atom/1`) or an in-file provenance fact (developer-supplied structure). I therefore do **not** upgrade this to a finding — there is no attacker-controlled path through *this file's* API into the OTP sink. Downgraded from a hedge to an explicit, non-load-bearing scope statement.

2. **"Residual application-level risk."** An application that builds its `children`, `child_spec/2` arg, or `start_child/2` arg from untrusted input would let an attacker choose which module's `child_spec/1` runs and which MFA starts — arbitrary code execution. This is inherent to what a supervisor is and cannot be defended at this layer; it is a caller responsibility, correctly documented as such in the module. Not a defect in this file. This is a factual caveat verifiable in-file (the API accepts developer-provided module atoms by design) and is retained as a caveat, not upgraded.

## Did I verify this
- Re-read the dispatch and error paths (lines 800–880) and the full-file grep (`to_atom|concat|binary_to_term|apply|String.to|ensure_loaded|Module.`) — two benign matches only (`Module.has_attribute?` @532, `Code.ensure_loaded?` @857).
- The `is_atom/1` guards at 816 and 820 were read directly and are the load-bearing evidence for the dynamic-dispatch dismissal.
- I did **not** read the standing-context neighbour modules: they are compiler-front-end modules (`elixir_tokenizer` etc.) with no callgraph relationship to `supervisor.ex`; reading them would not verify any claim here. The reachable neighbours (`code.ex`, OTP `:supervisor`) either confirm the atom-only contract (`Code.ensure_loaded?`) or lie outside the repo tree (OTP) while being non-load-bearing for every verdict.
- No reproduction scripts were run: no candidate sink survived triage to a testable behavioural differential (this file is pure delegation — no parsing, decoding, or dual-implementation guard to push axis-carrying values through), so there is no axis a run could clear. Every "Not examined: none" above reflects that the file's own contribution to each sink was fully read, not that a differential was executed.

### Dropped in re-review
None. Every first-pass dismissal is retained; the two cross-file caveats are re-expressed as explicit non-load-bearing scope statements rather than dropped, and none was upgraded to a finding because no attacker-controlled input reaches any sink through this file's documented public API.
