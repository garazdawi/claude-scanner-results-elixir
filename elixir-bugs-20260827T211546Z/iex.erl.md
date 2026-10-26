# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/iex.erl
<!-- sha256: fd7de6d570d31be729f578a899970d75646d9e70ff64c37548f6d99707f97085 -->
<!-- tree: 3389f8405 2026-08-25 -->

# Review: `elixir/lib/elixir/src/iex.erl`

1. **Summary** — This is a small IEx boot shim whose two `shell/0` return shapes (`{iex,start,[...]}` and `{remote,Node,{iex,start,[...]}}`) match the `initial_shell` value consumed by `user_drv:start/1` at `elixir.erl:180`. I re-read the two neighbour modules the first pass depended on (`elixir_utils`, `elixir`) and traced the charlist→binary conversion to source. No correctness bug is reachable from its documented API. No findings.

2. **Bugfixes**

No findings.

Spec-totality note: the module declares no `-spec` for any of its four exports (`start/0`, `start/2`, `shell/0`, `sync_remote/2`), so there is no declared-vs-accepted gap to audit; each export was checked against its actual call contract:

- `start/0` → delegates to `start/2` with `{elixir_utils, noop, []}`; `elixir_utils:noop/0` confirmed exported and body-checked (`elixir_utils.erl:8` export, `77-78` returns `ok`), so the default MFA resolves. OK.
- `start/2` → spawns an unlinked boot process, waits for node start, sets `{binary,true},{encoding,unicode}`, invokes `'Elixir.IEx.Server':run_from_shell/2`; returns the spawned pid. No caller requires a specific return. OK.
- `shell/0` → returns an MFA tuple or `{remote,Node,MFA}`; consumed as `#{initial_shell => iex:shell()}` by `user_drv:start/1` (`elixir.erl:180`). OK.
- `sync_remote/2` → an exported public function referenced as the remsh shell MFA (`iex.erl:70`); performs the local/remote rendezvous. OK.

3. **Risks / caveats** — I read this file plus the two in-folder callgraph neighbours: `elixir.erl` (site `:180`, how `shell/0`'s result is consumed; `start_cli/0` at `:182`) and `elixir_utils.erl` (`noop/0` at `:77`, `characters_to_binary/1` at `:145-152`). One dependency remains genuinely out of scope: OTP `user_drv`'s exact acceptance set for `initial_shell` — `user_drv` is **not** in this folder's callgraph and I did not execute a live node to probe its accepted shapes, so that axis stays untried (see Contract-violation entry). `'Elixir.IEx.Server':run_from_shell/2` is Elixir source not present in this checkout (`lib/iex/**` absent) and is not a listed neighbour; `sync_remote/2` reachability does not depend on it — the function is **exported** (public surface) and referenced in an MFA literal at `iex.erl:70`, which establishes reachability at the file boundary. Whether the external consumer actually invokes that MFA is out-of-file behaviour of a non-neighbour and does not bear on any finding here.

### Ruled out

- **Operator / precedence typo**
  Reason: no compound guards or boolean expressions exist; the only conditionals are `case` on `get_remsh/1` (matched against `nil`) and pattern-matching function heads.
  Axes examined: `--remsh`/`--dot-iex` present-and-followed, present-as-last-element, absent | Not examined: none
  Basis: verified

- **Missing clause / catch-all**
  Reason: `get_remsh/1` and `get_dot_iex/1` each have a 2-element-prefix clause, a skip-one clause, and an empty-list base, covering every proper-list shape; `["--remsh"]` (flag with no value) falls to the skip clause → `[]` → `nil` rather than crashing.
  Axes examined: empty list, flag-as-last-element (arity-1 tail), flag-with-value, flag-absent, non-flag elements | Not examined: improper list (excluded — `init:get_plain_arguments/0` returns a proper list of charlists)
  Basis: verified

- **Wrong atom / wrong symbol**
  Reason: the only atom used cross-clause is `nil`, produced by `get_remsh/1`/`get_dot_iex/1` and matched in `shell/0`/consumed as `{dot_iex, nil}`; `{binary,true}`/`{encoding,unicode}`/`halt`/`on_eof` are the io/IEx option names.
  Axes examined: `nil` produce-vs-match parity, io:setopts keys | Not examined: none
  Basis: verified

- **State / accumulator threading error**
  Reason: `get_*` recursions pass the correct tail `T`; `sync_remote/2`/Parent exchange the same `Ref` and reply to the captured `self()`/`Other`; no rebinding of shadowed state variables.
  Axes examined: Ref threading through begin/done handshake, tail advancement | Not examined: none
  Basis: verified

- **Off-by-one / wrong arithmetic**
  Reason: the file contains no arithmetic, indices, lengths, or timeout math.
  Axes examined: (none applicable) | Not examined: none
  Basis: verified

- **Contract violation**
  Reason: `shell/0`'s two return shapes are the `mfa()` and `{remote,Node,mfa()}` variants; confirmed the consumer at `elixir.erl:180` passes the result as `#{initial_shell => iex:shell()}` to `user_drv:start/1`. No `-spec` contradicts. The exact set of `initial_shell` shapes `user_drv` accepts (including whether `Node` may be a charlist and whether a bare 3-tuple `{remote,Node,MFA}` is honoured) lives in OTP `user_drv`, which is not in this folder's callgraph.
  Axes examined: remsh vs non-remsh return shape at the `elixir.erl:180` call site | Not examined: exact `user_drv` acceptance set (OTP source, non-neighbour; not executed against a live node)
  Basis: asserted
  Depends on: OTP `user_drv` — `initial_shell` accepts `{M,F,A}` and `{remote, Node, {M,F,A}}` with `Node` a charlist.

- **Crash on stray / unexpected input**
  Reason: the external input is `init:get_plain_arguments/0` (proper charlist list), fully covered by the `get_*` clauses; no user term reaches an unguarded match. One boot-path note: a malformed `--dot-iex` value that is not valid Unicode makes `elixir_utils:characters_to_binary/1` (`elixir_utils.erl:148-151`) raise `Elixir.UnicodeConversionError`. This is an *intended* explicit error for a bad path argument, not an unguarded `badmatch`/`function_clause`, and it surfaces at boot with a typed exception — so it is a legitimate failure, not a stray-input crash bug.
  Axes examined: empty args, flag-at-end, repeated flags, non-flag noise, non-UTF-8 `--dot-iex` value | Not examined: none
  Basis: verified

- **Resource leak on error path**
  Reason: the remsh `Parent` is created with `spawn_link`, so it dies with the `user_drv` caller rather than leaking; the boot process in `start/2` is deliberately unlinked (it is the shell) by design.
  Axes examined: remote-MFA-never-runs (Parent blocked in receive), start_cli crash | Not examined: none
  Basis: verified

- **Silent corruption** — *Upgraded from asserted to verified in re-review.*
  Reason: the only data transformation is `elixir_utils:characters_to_binary/1` on the dot-iex path. I read its body (`elixir_utils.erl:145-152`): it returns binaries unchanged, and for charlists calls `unicode:characters_to_binary/1`, returning the full binary on success and raising `Elixir.UnicodeConversionError` on `{error,…}`/`{incomplete,…}`. There is no truncation or lossy branch — a partial/invalid input errors out rather than being silently shortened.
  Axes examined: dot-iex value passthrough, binary-input shortcut, error/incomplete branches | Not examined: none
  Basis: verified

- **Infinite loop / hang**
  Reason: `sync_remote/2`'s unguarded `receive {done,Ref}` and Parent's `receive {'begin',Ref,_}` are the intended local/remote rendezvous; both counterparties are linked into the same boot lineage, so a peer death propagates rather than hanging a live system indefinitely.
  Axes examined: begin/done handshake ordering, peer-death propagation | Not examined: none
  Basis: verified

- **Quadratic-where-linear**
  Reason: `get_*` recursions are linear single-pass scans over a short argv; no accumulator concatenation.
  Axes examined: argv scan | Not examined: none
  Basis: verified

- **Repeated work**
  Reason: no compiled patterns, no per-iteration recompilation; the file runs once at boot.
  Axes examined: boot-time one-shot | Not examined: none
  Basis: verified

- **Dead / unreachable code**
  Reason: all four exports are reachable. `shell/0` is called from `elixir.erl:180`; `start/2` from `shell/0`'s MFA and `start/0`; `sync_remote/2` is a public export referenced as the remsh MFA literal (`iex.erl:70`) — being exported, it is part of the callable surface regardless of the external consumer. `start/0` is a documented public entry.
  Axes examined: export reachability, MFA-literal reference | Not examined: none
  Basis: verified

- **Documented behaviour mismatch**
  Reason: the module's only prose is the manual-test comment block (test recipes, not behavioural claims); no `@doc`/`-spec` asserts a property the code could contradict.
  Axes examined: each comment recipe vs code path (remsh, --dot-iex, -e boot) | Not examined: none
  Basis: verified

- **Race condition / shared mutable state**
  Reason: no shared ETS/`system_flag`/global state is touched; the Parent/sync_remote handshake uses a fresh `make_ref/0` per invocation as the correlation token.
  Axes examined: concurrent shell boots (distinct Refs) | Not examined: none
  Basis: verified

- **Wrong scope / order of operations**
  Reason: `Parent` is spawned before the MFA that references it is constructed, and `io:setopts` runs after node-started confirmation and before `run_from_shell`; ordering matches intent.
  Axes examined: spawn-before-reference, wait-before-setopts | Not examined: none
  Basis: verified

- **Encoding / unicode mishandling** — *Upgraded from asserted to verified in re-review.*
  Reason: `io:setopts([{binary,true},{encoding,unicode}])` is set before the shell runs; dot-iex conversion delegates to `elixir_utils:characters_to_binary/1`, whose body (`elixir_utils.erl:145-152`) performs correct UTF-8 charlist→binary conversion via `unicode:characters_to_binary/1` and raises `Elixir.UnicodeConversionError` on invalid/incomplete sequences rather than mis-encoding. No byte/codepoint confusion in this file.
  Axes examined: io encoding setup, dot-iex charlist conversion, invalid/incomplete UTF-8 handling | Not examined: none
  Basis: verified

- **Unclassified**
  Reason: no file-specific one-off sins remain; the remsh handshake, though intricate, is internally consistent and matches the documented cross-node argument-processing requirement (comment #3).
  Axes examined: remsh cross-node handshake semantics | Not examined: none
  Basis: verified

### Did I verify this?

- All `verified` entries were traced against the file's clauses with the named axes tried and `Not examined: none`.
- Two entries that were `asserted` in the first pass are now `verified`, on read evidence from the in-folder neighbour:
  - **Silent corruption** and **Encoding / unicode mishandling** — both depended on `elixir_utils:characters_to_binary/1`. I read `elixir_utils.erl:145-152`: it is a faithful charlist→binary conversion that raises on error and never truncates. The dependency is discharged; no assumption remains.
- One entry stays `asserted` and I keep it so rather than upgrade on a re-read:
  - **Contract violation** — depends on OTP `user_drv` accepting `{M,F,A}` and `{remote,Node,{M,F,A}}` (Node a charlist). `user_drv` is not a callgraph neighbour and I did not execute a live node to probe its acceptance set, so the `Not examined` axis is real. Reading `elixir.erl:180` confirmed only *how* the value is handed off (`initial_shell` map key), not *what shapes* `user_drv` accepts. Downgrade-safe status retained.

### Dropped in re-review

None. Every ruled-out entry from the first pass is preserved; two were strengthened (asserted → verified) and none were weakened or removed.

### Hedge disposition

The first pass carried two hedge phrases ("I assumed", "I assume"):
- *"I assumed it invokes the passed MFA, which is what makes `sync_remote/2` reachable"* — reframed without assumption: `sync_remote/2` is an **exported** function referenced by an MFA literal at `iex.erl:70`, so it is reachable at the file boundary independent of any external consumer. `'Elixir.IEx.Server':run_from_shell/2` is not present in this checkout and is not a callgraph neighbour, and no iex.erl finding rides on its behaviour; no low-confidence finding is warranted.
- *"I assumed [`characters_to_binary`] performs correct conversion"* — verified by reading `elixir_utils.erl:145-152`; the two dependent entries are now `verified`.
