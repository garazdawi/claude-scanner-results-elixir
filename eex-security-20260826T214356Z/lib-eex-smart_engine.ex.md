# /Users/lukas/code/claude-scanner/elixir/lib/eex/lib/eex/smart_engine.ex
<!-- sha256: 773cb44eb882ab501b13cafe9832049f803905327e8c318efa6555cc832e46ef -->

# /Users/lukas/code/claude-scanner/elixir/lib/eex/lib/eex/smart_engine.ex
<!-- sha256: 773cb44eb882ab501b13cafe9832049f803905327e8c318efa6555cc832e46ef -->

_Re-analysed on 2026-08-26_

Confirmed empirically, with the cross-file hedges from the first pass now resolved by reading `EEx.Engine` (`elixir/lib/eex/lib/eex/engine.ex`) and by executing the relevant axes on the local Erlang/OTP 29 toolchain.

**Phase 1 — inventory of security-relevant sinks in this file.** The file is a 62-line `EEx.Engine` behaviour implementation. The delegated callbacks (`init`, `handle_body`, `handle_begin`, `handle_end`, `handle_text`) are pure `defdelegate` forwards to `EEx.Engine` (lines 43–55). The only logic is `handle_expr/3` (lines 58–61), whose two operations are:
- `Macro.prewalk(expr, &EEx.Engine.handle_assign/1)` — AST walk with a cross-file callback.
- `EEx.Engine.handle_expr(state, marker, expr)` — cross-file call.

No `Code.eval*`, no `String.to_atom`/`Module.concat`, no `binary_to_term`, no file/path/shell/archive/network/crypto sink is present in this file.

**Phase 2 — per-sink judgment (cross-file callees now read, not assumed).**
- `handle_assign/1` (engine.ex:128–135) matches only `{:@, meta, [{name, _, atom}]}` with `is_atom(name) and is_atom(atom)` — `name` is already an atom in the parsed AST — and rewrites to `EEx.Engine.fetch_assign!(var!(assigns), name)`. The assign *value* is not present in this AST node at rewrite time; it is fetched at runtime via `Access.fetch` (engine.ex:141). No data-to-atom conversion.
- `fetch_assign!/2` (engine.ex:140–156) on a missing key does **not** raise: it emits `IO.warn` and returns `nil` (TODO to raise on v2.0). This is a resilience choice, not a sink; it processes the assign *value*, not the key-space, through `Access`.
- The one data-independent atom sink in the chain, `String.to_unsafe_atom("arg#{vars_count}")`, lives in `EEx.Engine.handle_expr(state, "=", …)` (engine.ex:198). `vars_count` is an integer counter incremented per `<%= %>` expression (engine.ex:210), bounded by template size; the generated string reuses the same atom (`arg0`, `arg1`, …) across compilations.
- `marker` is rejected for unsupported values `"/"` and `"|"` in `EEx.Engine.handle_expr/3` (engine.ex:218–221), raising `EEx.SyntaxError`.
- `check_state!/1` (engine.ex:223–228) is the only defensive gate and lives in `EEx.Engine`, not this file.

## Summary
`EEx.SmartEngine` is a thin `EEx.Engine` behaviour implementation whose only logic prewalks `@assign` AST rewriting before delegating to the base engine; it introduces no in-file security sink. All cross-file assumptions from the first pass were verified by reading `EEx.Engine` and by executing the atom/marker axes.

## Security
No findings

### Ruled out

- **Code execution / SSTI (template is code) — Confidence: High.** `handle_expr` receives the quoted AST of a template expression, produced upstream by `Code.string_to_quoted!` in `EEx.Compiler`. Untrusted templates are RCE by EEx's documented, by-design contract (the caller must treat the template as trusted). This file adds no new eval/compile sink; the injection surface is identical with or without the assigns rewrite. By-design, out of scope.
  - Axes examined: presence of eval/compile sink in this file (none); whether the assigns rewrite widens the surface (it does not — it only substitutes `@x` for a `fetch_assign!` call). | Not examined: none.

- **Atom-table exhaustion (CWE-770) — Confidence: High.** `EEx.Engine.handle_assign/1` (engine.ex:128) reuses the already-parsed `@name` atom and never converts assign *values* to atoms. The `String.to_unsafe_atom("arg#{vars_count}")` at engine.ex:198 is keyed on an integer counter, not data, and dedups to a bounded `arg{N}` set.
  - Axes examined (all executed on OTP 29):
    1. *Hostile assign value → atoms:* pushed a novel per-run string (`"hostile_atom_#{system_time}_zzz"`) as an assign value through `EEx.Engine.fetch_assign!/2` → **0 atoms created**, and the string is still not an existing atom afterward (`String.to_existing_atom` raises).
    2. *AST rewrite of `@foo`:* `Macro.prewalk(quote(@foo), &handle_assign/1)` created 7 atoms one time — these are the fixed literal atoms in the generated `EEx.Engine.fetch_assign!(var!(assigns), :foo)` node (module/function/var aliases), input-independent constants, not data-derived.
    3. *`vars_count` atom bound:* 1000 `handle_expr(_, "=", _)` calls all pinned at `vars_count=0` created **0 atoms** (the `arg0` atom is reused).
  - Not examined: none.

- **Resource consumption — Confidence: High.** `Macro.prewalk/2` is linear in AST node count, bounded by the (already-parsed) template size; no attacker-controlled amplification is introduced in this file.
  - Axes examined: per-node work of `prewalk` (constant — a single pattern-match in `handle_assign`); superlinear expansion (none — one node maps to one node). | Not examined: none.

- **Delegated callbacks (`init`/`handle_body`/`handle_begin`/`handle_end`/`handle_text`) — Confidence: High.** Pure `defdelegate` forwards (lines 43–55) to `EEx.Engine` with no in-file logic or sink. The base implementations (engine.ex:159–192) only manipulate the `%{binary, dynamic, vars_count}` accumulator and guard shape via `check_state!/1`.
  - Axes examined: presence of in-file logic (none); shape of callees read in engine.ex. | Not examined: none.

- **`marker` passthrough — Confidence: High.** `marker` originates in the EEx tokenizer (internal), not external data, and is forwarded to `EEx.Engine.handle_expr/3`. Verified by execution that markers `"/"` and `"|"` raise `EEx.SyntaxError` (engine.ex:218–221); `""` and `"="` are the only accepted values. No trust boundary crossed in this file.
  - Axes examined: rejection of unsupported markers (executed — `"|"` raised `EEx.SyntaxError`); origin of `marker` (tokenizer, per standing context). | Not examined: none.

## Risks / caveats
- The substantive assign-handling behavior (`handle_assign`, `fetch_assign!` with its non-raising `IO.warn`-on-missing-key at engine.ex:148, and the base `handle_expr` with `String.to_unsafe_atom`) lives in `EEx.Engine`. These were read and their security-relevant axes executed for this re-review; any defect ownership for them belongs to `engine.ex`, reviewed as its own file.
- The only defensive gate in the call chain (`check_state!/1`, engine.ex:223) is in `EEx.Engine`, not this file; nothing in `smart_engine.ex` is dead defensive code.
- A malicious custom engine substituted for `EEx.Engine` is outside this file's control and not considered.

## Did I verify this?
- **Read `EEx.Engine` (engine.ex).** Confirmed `handle_assign/1`, `fetch_assign!/2`, `handle_expr/3` (incl. `String.to_unsafe_atom` and marker rejection), and `check_state!/1` bodies — resolving every "lives in another file / out of scope" hedge from the first pass into a read-and-verified claim.
- **Executed on OTP 29** (`elixir t.exs`): hostile assign value → 0 atoms and remains a non-atom; `@foo` rewrite creates only fixed input-independent atoms; 1000 `handle_expr` calls at fixed `vars_count` → 0 atoms; marker `"|"` raises `EEx.SyntaxError`.
- Not run end-to-end through `EEx.eval_string`, because the two operations of this file are exactly the two functions exercised above in isolation; the eval path only adds the (trusted-by-contract) compile of the template.

## Upgraded from hedged assumption in first pass
None. Every first-pass hedge ("lives in `EEx.Engine`", "out of scope for this file", "verified 0 atoms" without cited execution) was resolved by **verification** (reading the callee and executing the axis), so each dismissal moved from *asserted* to *verified* rather than up to a finding. No assumption remained unverified, so no upgrade to a low-confidence finding was warranted.

## Dropped in re-review
None. All first-pass ruled-out entries are retained, now with executed evidence and `Not examined: none`.
