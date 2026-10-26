# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/autocomplete.ex
<!-- sha256: 4725cb1ead57e722ffb4650137c9d5b91d8da0a24e653fd2d3f23cb375ca168f -->

# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/autocomplete.ex
<!-- sha256: 4725cb1ead57e722ffb4650137c9d5b91d8da0a24e653fd2d3f23cb375ca168f -->

_Re-analysed on 2026-08-27_

The complete caller set is now confirmed by reading, not assumed: `server.ex` (registers `expand`/`remsh` as the terminal's `expand_fun`), `helpers.ex:1079-1080` (the operator-invoked `exports` helper), `remsh/1`'s own erpc, and tests. No dependency-, archive-, build-, or network-driven caller exists.

## Summary

`IEx.Autocomplete` mints permanent atoms and lists directories from the completion fragment, but every such sink is reachable only through the interactive shell's `expand_fun` (a `@moduledoc false` module driven by the trusted operator's keystrokes), so no untrusted-data path reaches them. All three cross-file hedges from the first pass are now verified against the neighbour source and none of them opens an external reachability path.

## Security

No findings

### Ruled out

- **Atom-table exhaustion via `List.to_unsafe_atom` / `String.to_unsafe_atom` (lines 103, 119, 241, 252, 261) and `Module.concat` (lines 496–500).** *Verified* that novel fragments create permanent atoms (`elixir ac_test2.exs` → `atom delta over 1200 novel fragments = 1319`; `zzznovelfun177 now an atom? true`). This is the textbook CWE-770 sink — but the value enters only via `IEx.Autocomplete.expand/2`. The wiring is now *read, not assumed*: `server.ex:353-358` sets `expand_fun` to `&IEx.Autocomplete.expand/1` (local) or `IEx.Autocomplete.remsh(node())` (remote), and `server.ex:135` registers it with `:io.setopts(expand_fun: expand_fun)`, so the sole feed is the reversed terminal line buffer on TAB. The module is `@moduledoc false`; no caller forwards network/disk/dependency/archive/build data into it. The interactive operator can already create atoms (or run anything) by pressing Enter — autocomplete is strictly weaker than eval. Out of scope ("user runs code in their own shell"). **Confidence: high** (reachability verified by reading `server.ex`).
- **`String.to_unsafe_atom(full_name)` in `expand_structs` (line 342) and `usable_as_unquoted_module?` (line 611).** The strings come from `match_modules`, i.e. `Atom.to_string/1` of already-loaded modules (`:erlang.loaded()`, application module lists, lines 614-642), so the atom already exists; conversion mints nothing new, matching the in-code comments. **Confidence: high** (data source is in-file).
- **Filesystem access in `expand_path` / `ls_prefix` (`File.ls`, `File.dir?`, `Path.join`, `Path.dirname`, `Path.basename`, lines 880-917).** Read-only directory enumeration of a path fragment the operator typed inside quotes (`path_fragment/2`). No write/delete/link, no confinement need; `..` merely lists another directory the operator could `File.ls` directly. Trusted-operator input. **Confidence: high** (in-file).
- **Module dispatch via `exports/1`, `alias.__struct__()` (line 362), `Code.ensure_all_loaded` (line 346), `mod.__info__/1`.** The module atoms resolve from operator-typed alias/dot fragments and are gated by `ensure_loaded?`/`function_exported?`; the operator controls them and could invoke the modules directly. The one non-shell caller of `exports/1` is `helpers.ex:1079-1080` (`def exports(module \\ Kernel)`), an operator-invoked helper documented at `helpers.ex:34` — *verified by reading*, no untrusted feed. **Confidence: high**.
- **`remsh/1` erpc to a remote node (line 52).** The `node` and the fragment `e` both originate on the operator's side; the remote node shares the distribution cookie and is fully trusted per the folder threat model. Within-cluster, out of scope. **Confidence: high**.
- **`Code.Fragment.*` / `Macro.path` container parsing (`container_context/2`, lines 401-437).** Operates on the typed line; parse failures are handled (`{:error, _} -> nil`, line 434). Operator input. **Confidence: high** (in-file).

## Risks / caveats

- **Atom growth is real but operator-scoped.** Every distinct completion fragment permanently grows the atom table (verified above). Harmless interactively, but `IEx.Autocomplete.expand/2` is *not* safe to expose to untrusted input — anyone repurposing this `@moduledoc false` module behind a network-facing completion service would hand a remote client an atom-exhaustion DoS. Worth a one-line "internal, do not feed untrusted input" note if the module is ever surfaced.
- **Robustness (non-security), now cross-file-verified:** a completion token longer than the 255-byte atom limit raises `SystemLimitError` from `List.to_unsafe_atom` (*verified*: `String.duplicate("a",20000)<>"("` → `{:error, :system_limit}`). The `remsh` path swallows this (`catch _, _ -> {:no, …}`, autocomplete lines 51-55). The local path is `&IEx.Autocomplete.expand/1` — I *read* `server.ex:357` and confirmed it is passed to `:io.setopts` (line 135) **without** any try/catch wrapper, so such a fragment lets the exception escape into the IO/edlin server. Self-inflicted by the operator only; no external reachability. Not a security finding.
- **`IEx.Broker.evaluator/1` MatchError, now verified in `broker.ex` (lines 37-42) — Upgraded scope of evidence from hedged assumption in first pass.** The first pass asserted "that behavior lives in broker.ex, outside this file" without reading it. I have now read it:
  ```elixir
  def evaluator(pid \\ shell()) do
    if pid do
      {:dictionary, dictionary} = Process.info(pid, :dictionary)   # nil for a dead pid → MatchError
      {dictionary[:evaluator], pid}
    end
  end
  ```
  Confirmed: a non-nil-but-dead `shell` pid makes `Process.info(pid, :dictionary)` return `nil`, and the hard match `{:dictionary, dictionary} = nil` raises `MatchError`. This module's `imports_from_env`/`aliases_from_env`/`variables_from_binding`/`value_from_binding` (lines 830-863) call `IEx.Broker.evaluator(shell)` inside a `with ... else _ -> []`, but `with` does not trap raised exceptions, so a `MatchError` would propagate out of the local `expand/1` (which, per above, is unwrapped). In production `shell` is `:shell.whereis()` or `IEx.Broker.shell()`, which yield a live pid or `nil` (the `nil` branch returns `nil`, handled by the `with` else). So this is reachable only with a stale/dead shell pid — operator- or test-scoped, no external reachability. Kept as a robustness caveat, **not** a security finding. **Confidence: this is a non-finding; high confidence it is not externally reachable** (the crashing line is confirmed in `broker.ex`, and the only production pid sources are `:shell.whereis()`/`IEx.Broker.shell()`).

## Did I verify this?

- **`server.ex` autocomplete wiring** — *Read.* `server.ex:353-358` builds `expand_fun` (local `&IEx.Autocomplete.expand/1` vs remote `IEx.Autocomplete.remsh(node())`); `server.ex:135` registers it via `:io.setopts(expand_fun: expand_fun)`; `server.ex:143` swaps to a no-op during eval. Confirms the sole feed is the terminal line buffer and that the local path is unwrapped. (Previously hedged as "wired to the shell exclusively" — now read.)
- **`helpers.ex` exports caller** — *Read.* `helpers.ex:1079-1080` `def exports(module \\ Kernel)` calls `IEx.Autocomplete.exports(module)`; operator-invoked helper, no untrusted feed. (Previously cited by line number without reading — now read.)
- **`broker.ex` evaluator/1 MatchError** — *Read.* `broker.ex:37-42`, crash line confirmed; reachability bounded to dead/stale shell pids. (Previously asserted "outside this file" — now read.)
- **Atom minting on novel fragments** — *Ran* (`ac_test2.exs`): +1319 atoms over 1200 novel fragments; `zzznovelfun177` became an atom.
- **255-byte token → `SystemLimitError`** — *Ran*: `String.duplicate("a",20000)<>"("` → `{:error, :system_limit}`.
- **Not run:** I did not execute the dead-shell-pid MatchError path end-to-end through `expand/1` in a live IEx; the crash line is confirmed statically in `broker.ex` and the conclusion (operator/test-scoped, not externally reachable) does not depend on the untried execution.

## Dropped in re-review

None. Every ruled-out entry and caveat from the first pass is preserved; the three cross-file hedges ("wired to the shell", the `helpers.ex` caller, and the "outside this file" `broker.ex` MatchError) were converted from assertions into read-verified statements, and none of them rose to a security finding.
