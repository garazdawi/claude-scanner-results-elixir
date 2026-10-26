# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/node.ex
<!-- sha256: 88a4ee2261c20036a1c1d9cf975c3925bf707c8b6867295752748d52ca0c6d0c -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/node.ex
<!-- sha256: 88a4ee2261c20036a1c1d9cf975c3925bf707c8b6867295752748d52ca0c6d0c -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T13:40:00Z (first pass 2026-08-28T12:05:27Z)_

## Summary

`Node` is a thin delegation layer over Erlang's distribution BIFs (`:net_kernel`, `:erlang`, `:net_adm`). Every argument reaching a sink is an already-typed atom/list/fun/opts term supplied by the caller; no external bytes are parsed or converted, no string→atom conversion occurs, and the only non-forwarding logic in the file (`start/2` clause dispatch + `Map.new(opts)`) was executed and shown to be a deterministic rekey with no atom creation. No security defect is reachable through the documented public API. **No findings** (unchanged from first pass; the first-pass scope hedges are resolved below with executed/inspected evidence rather than assumption).

### Re-review disposition of first-pass hedges

The first-pass reviewer flagged three "scope-boundary / out-of-scope-for-this-file" hedges (the three "Risks / caveats" bullets). Resolution:

- **Hedge 1 — "the machinery it delegates to lives in `:net_kernel`/`:erlang`/ERTS, out of scope."** *Verified, not assumed.* The delegation targets are OTP modules that physically live outside this repository: `net_kernel.erl` is at `/usr/local/lib/erlang/lib/kernel-11.0.2/src/net_kernel.erl` (OTP 29), and `:erlang`/`:net_adm` are ERTS/kernel BIFs. They are not files in this folder's callgraph (the standing-context neighbours are all `elixir_*` compiler-src modules, none of which `node.ex` calls). This is therefore a genuine external boundary, confirmed by locating the source outside the repo — not an un-read neighbour I skipped. It stays as verified context, not a finding: nothing in `node.ex` becomes defective because the sink is elsewhere.
- **Hedge 2 — "distributed Erlang trusts any node with the correct cookie (documented threat model)."** This is a documented ERTS platform property, not an assumption about an un-read file, and `node.ex` neither strengthens nor weakens it. It cannot be upgraded to a `node.ex` finding because the trust decision is not implemented here. Retained as context.
- **Hedge 3 — "no behavioral reproduction was required."** Re-review *did* exercise the toolchain (OTP 29 / Elixir) against the only in-file logic; there remains no positive finding to reproduce. Retained, now backed by the executed dispatch test below.

None of the three hedges was an unverified behavioral assumption about a repo neighbour, so none is upgraded to a low-confidence finding. Reading `net_kernel.erl` was available but is not cited as clearing anything — per the re-review rule, reading a neighbour tells you what a guard does, not which inputs disagree with it; the no-finding conclusion for `node.ex` rests only on the in-file fact (verified by full read + execution) that arguments are already-typed and forwarded unchanged.

## Security

`No findings`

I read the file in full (356 lines) and traced every call site. Each public function forwards caller-supplied, already-typed arguments to an Erlang BIF; none parse network/file/archive data, convert strings to atoms, evaluate code text, or touch the filesystem/shell. A whole-file grep for the classic sink shapes (`to_atom`, `binary_to_term`, `Code.*`, `System.*`, `:os.*`, `Module.concat`, `eval`, `Regex`, `Path.*`, `File.*`, `EEx`, `:erl_tar`, `:zip`) returned a single hit — inside a doc comment, not code.

### Ruled out

- **`spawn/2..5`, `spawn_link/2..4`, `spawn_monitor/2..4` — remote code execution (Code-execution class).** *Confidence: high.* These run a caller-supplied fun/MFA on a remote node; that is their documented purpose. The fun/MFA is not derived from external input within this file, and the remote node is reachable only after cookie-authenticated distribution is established (the authentication is enforced in `:net_kernel`/ERTS, outside this repo). Running attacker-controlled code as the fun would be caller misuse, not a defect in `node.ex`. *Basis: full read; the `fun`/`module,fun,args` values originate from the caller in every clause (lines 234–334), forwarded unchanged to `:erlang.spawn*`. Axes examined: argument provenance (caller-supplied), in-file transformation (none). Not examined: none — no argument is transformed in this file, so there is no in-file axis left to try; the remote-side authentication behavior is not implemented here and is not an axis of `node.ex`.*
- **Atom creation / atom-table exhaustion (CWE-770).** *Confidence: high.* No `String.to_atom`, `List.to_atom`, `:erlang.binary_to_atom`, or `Module.concat` anywhere; `node`/`cookie` are atoms per the API, and `Map.new(opts)` only rekeys an existing keyword list. *Basis: whole-file grep (only a doc-comment match) + full read + executed test: `Map.new([a: 1, a: 2]) => %{a: 2}` (last-write-wins rekey of pre-existing atoms; no bytes→atom path). Axes examined: sink presence (none), `Map.new` key-creation behavior (rekey only, executed). Not examined: none.*
- **`start/2,3` clause dispatch — unexpected clause selection.** *Confidence: high. (New in re-review; substantiates the previously asserted "dead/unreachable defensive code" claim with execution.)* The guards `is_list(opts)` (line 76) and `is_atom(type)` (line 81) partition inputs deterministically. *Basis: executed the exact guards/bodies: keyword list and `[]` → clause 1 (`Map.new`); `:longnames` and `nil` → clause 2 (legacy `[name, type, 15000]`). No overlap or shadowing; no security-relevant surprise in either branch. Axes examined: list vs atom second arg, empty list, `nil`-as-atom. Not examined: none.*
- **`get_cookie/0` and `set_cookie/1,2` — secret handling (Cryptography class).** *Confidence: high.* The magic cookie is a distribution secret, but these are the documented getter/setter; the file does not log it, transmit it, or compare it non-constant-time. Returning/setting the cookie is the function's contract, not a leak. *Basis: lines 343–355 forward directly to `:erlang.{get,set}_cookie` with no additional handling. Axes examined: logging (none), transmission (none), comparison (none — no comparison performed in-file). Not examined: none.*
- **`connect/1`, `ping/1`, `disconnect/1` — network target from external data (Network class).** *Confidence: high.* The target is a caller-supplied atom; the actual sink is Erlang's distribution, gated by cookie authentication and implemented in `:net_kernel`/`:erlang` (outside this repo). `node.ex` only forwards the atom. *Basis: lines 194, 209, 222 — single-BIF forwards, no in-file transformation. Axes examined: argument provenance (caller atom), in-file transformation (none). Not examined: none.*
- **Command execution (Command-execution class).** *Confidence: high.* `System.cmd("epmd", ["-daemon"])` appears only in the `start/2` docstring (line 27) as guidance; it is never executed by this module. *Basis: grep located the single occurrence inside the `@doc` string. Axes examined: code vs doc context (doc only). Not examined: none.*

## Did I verify this

- **Read:** full file (356 lines).
- **Located (external boundary):** `net_kernel.erl` at `/usr/local/lib/erlang/lib/kernel-11.0.2/src/net_kernel.erl` — confirming the delegation targets live in the OTP install, not this repository. `Map` and other Elixir stdlib callees are likewise not present in this repo subtree (this folder is `lib/elixir/lib` limited to `node.ex` for the callgraph in scope). No `elixir_*` compiler-src neighbour from the standing context is called by `node.ex`.
- **Executed (Elixir on OTP 29):** the `start/2` clause dispatch and `Map.new/1` rekey behavior — results inline above. Confirmed deterministic dispatch and no atom creation.
- **Not executed / not needed:** the remote-side distribution/cookie-authentication path in `:net_kernel`/`:erlang`. This is not an axis of `node.ex` (no code here implements or depends on that behavior for its no-finding verdict); the file merely forwards already-typed terms. Reading it would not clear any `node.ex` axis, so it is not cited as verification.

## Dropped in re-review

Nothing dropped. All six first-pass ruled-out entries are retained (one — `start/2` dispatch — was promoted from an asserted "dead code" note to an executed-evidence entry, and one entry, atom-table exhaustion, gained an executed `Map.new` test). No first-pass finding existed to drop, and no ruled-out entry was upgraded to a finding: each rests on an in-file fact (typed-argument forwarding, no sink present) verified by full read and, where logic existed, by execution — not on an unverified assumption about a neighbour file.
