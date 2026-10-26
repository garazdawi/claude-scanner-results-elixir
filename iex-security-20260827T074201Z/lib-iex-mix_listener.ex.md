# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/mix_listener.ex
<!-- sha256: 5fdd81fc13b4f98f8e3ff209464e7b001a435239ae38a36bba4f7f9c54756cf9 -->

# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/mix_listener.ex
<!-- sha256: 5fdd81fc13b4f98f8e3ff209464e7b001a435239ae38a36bba4f7f9c54756cf9 -->

_Re-analysed on 2026-08-27_

## Summary
`IEx.MixListener` performs irreversible `:code.purge`/`:code.delete` on module names taken from an inbound `{:modules_compiled, info}` message. The sink is destructive (verified: unloads arbitrary in-memory modules and kills processes running a targeted module) and a malformed message body crashes the GenServer (verified). Whether an *untrusted* party can deliver that message depends on the `Mix.Sync.PubSub`/`Mix.PubSub` transport, which is **not in this checkout** and could not be verified or re-run in this pass — that reachability premise is therefore asserted, not verified, this time.

## Security

#### `CSCAN-mix_listener-untrusted-code-purge`
- **Finding ID:** `CSCAN-mix_listener-untrusted-code-purge`
- **Severity:** Medium
- **Confidence:** medium
- **Location:** `mix_listener.ex:45-63` (`handle_info/2`) → `mix_listener.ex:69-74` (`purge_modules/1`)
- **Class:** Deserialisation / Shared mutable state (destructive code-unload from an untrusted message)
- **Issue:** `handle_info({:modules_compiled, info}, state)` treats the whole `info` map as authoritative. The only gate is `info.os_pid == System.pid()` (line 46), a plain string comparison an attacker sets to any other value. It then reads `info.modules_diff.changed`/`.removed` (line 51) and, under `auto_reload?`, feeds them straight into `purge_modules/1`, which runs `:code.purge(module)` then `:code.delete(module)` (lines 71-72) for every module the message names. Nothing in this file confines those modules to ones the project actually compiled or owns.
- **In-file behaviour (verified by execution, OTP 29):** the sink is destructive on an untrusted-shaped message. Sending `{:modules_compiled, %{os_pid: "0", modules_diff: %{changed: [Victim], removed: []}}}` (os_pid mismatching `System.pid()`) unloaded the target: `is_loaded before: {:file, []}` → `is_loaded after: false`; a subsequent call raised `UndefinedFunctionError (module Victim is not available)`. The listener stayed alive. Two such messages targeting a live worker's module killed it: `worker alive before: true` → `after 1st: true` (delete moves current→old) → `after 2nd: false` (purge kills the process lingering in old code). This confirms the destructive-endpoint claim without relying on any other file.
- **Reachability premise (asserted — could NOT be verified this pass):** the message is documented to be delivered over `Mix.Sync.PubSub`/`Mix.PubSub`. Those modules are **not in this checkout** (`lib/` contains only `iex/`), and `IEx.MixListener.start_link/1` is never called anywhere under `lib/iex` (grep: the only in-tree references are the definition and `helpers.ex:147` calling `purge/0`). `IEx.App.start/2` (`app.ex:15`) supervises only `[IEx.Config, IEx.Broker, IEx.Pry]` — **not** `MixListener` — confirming the process and its message sender are both wired up outside this checkout by Mix. I therefore cannot verify, in this pass, that a *non-project / different-local-user* peer can forge and deliver `{:modules_compiled, ...}`, nor that the wire decode omits `[:safe]`. The prior report's end-to-end forgery evidence relied on the real Mix transport, which is not reproducible here. This premise is what keeps the finding at **medium** confidence and severity Medium rather than High.
- **Impact (conditional on the reachability premise):** a local process able to deliver the message into a running `iex -S mix` node can (a) unload arbitrary in-memory modules → `UndefinedFunctionError` on code that cannot reload from disk, and (b) kill any process by naming its module (double-purge). Repeatable → sustained DoS / crash-looping. Not RCE (payload is a non-executable term). If the transport turns out to be authenticated/same-user-only, the practical exposure collapses to a same-privilege actor and this is not a boundary crossing.
- **Evidence:** in-file destructiveness `verified` (executed, OTP 29, outputs above). Transport reachability `asserted` (source not in checkout; not re-runnable here).
- **Suggested fix:** durable fix in the delivery layer (authenticate `Mix.Sync.PubSub` senders; decode with `[:safe]`) — outside this file and outside this checkout. In-file defense-in-depth: before purging, drop any module whose `:code.which/1` object path is not under the running project's build/consolidation path, so a message cannot target arbitrary system/other-app modules or kill unrelated processes.

#### `CSCAN-mix_listener-malformed-diff-crash`
- **Finding ID:** `CSCAN-mix_listener-malformed-diff-crash`
- **Severity:** Low
- **Confidence:** high
- **Location:** `mix_listener.ex:46,51` (`handle_info({:modules_compiled, info}, state)`)
- **Class:** Resource consumption / robustness (crash on malformed input)
- **Issue:** `info.os_pid` (line 46) and the strict match `%{changed: changed, removed: removed} = info.modules_diff` (line 51) assume a well-formed `info`. A `modules_diff` missing `:changed`/`:removed`, a non-map `modules_diff`, or a missing `:os_pid` raises `MatchError`/`KeyError` and kills the GenServer. The catch-all `handle_info(_message, state)` (line 65) only covers non-`:modules_compiled` messages; a `:modules_compiled` with a malformed body still reaches line 51.
- **Impact:** whoever can deliver the message can crash the supervised listener at will. Escalation past a `:one_for_one` restart intensity into disabling Mix's auto-reload subsystem is **asserted** — the supervisor (`Mix.PubSub.ListenerSupervisor`) is not in this checkout, so the restart-intensity claim could not be verified this pass. The crash itself is verified.
- **Evidence:** `verified` (executed, OTP 29). Sending `{:modules_compiled, %{os_pid: "0", modules_diff: %{}}}` produced `** (MatchError) no match of right hand side value: %{}` at `lib/iex/mix_listener.ex:51` and a `{:DOWN, ...}` with reason `{:badmatch, %{}}`; the listener terminated. (The os_pid path also exercised: the message passed the `os_pid == System.pid()` check because `"0"` ≠ the runtime pid, reaching line 51.)
- **Suggested fix:** match defensively and ignore off-shape bodies — e.g. `case info do %{os_pid: os, modules_diff: %{changed: c, removed: r}} -> ...; _ -> {:noreply, state} end`.

### Ruled out
- **Atom-table exhaustion in this file** — `Axes examined: sink inventory (whole file), value flow of `changed`/`removed` | Not examined: none.` The module contains no `String.to_atom`/`binary_to_atom`/`List.to_atom` sink (verified by reading all 76 lines); `changed`/`removed` are iterated as-is into `:code.purge`/`:code.delete`, which take existing atoms and create none. Any new-atom-on-the-wire creation would occur in the transport decode (`Mix.Sync.PubSub`), which is not in this checkout — not attributable to this file.
- **Unbounded `to_purge` growth** (`Enum.into`, lines 58-59) — `Axes examined: element type (atoms), set semantics | Not examined: none for this file.` The `MapSet` holds distinct atoms, bounded by the atom table; growth is capped by the same finite atom supply. No independent capability added here.
- **`os_pid` self-check as a security gate** (line 46) — `Axes examined: comparison operands (string == System.pid()), bypass by attacker-set value | Not examined: none.` It only suppresses self-compilations and is trivially set to any other string; neither a defense nor dead code — an optimization. Verified by execution above (`"0"` bypassed it and reached the else branch).
- **`purge/0` TOCTOU** (lines 25-26) — `Axes examined: whereis→call race, failure mode | Not examined: none.` `Process.whereis` then `GenServer.call`; a race only yields a benign caller-side exit, no security impact.

## Risks / caveats
- The load-bearing reachability premise (an *untrusted* peer can forge and deliver `{:modules_compiled, ...}`) lives entirely in the `Mix.Sync.PubSub`/`Mix.PubSub` transport, which is **absent from this checkout** — I could not Read it and there is no Mix here to re-run the end-to-end forgery. This file is only the destructive endpoint; its in-file behaviour (destructive purge/delete, crash on malformed body) is verified by execution, but the "untrusted sender" half is asserted. A reviewer with the Mix tree should confirm (i) senders are unauthenticated and cross-user reachable and (ii) the decode omits `[:safe]`; if either is false, both findings drop toward same-privilege / robustness-only.
- Reachability also requires the victim to be running IEx together with Mix's pubsub listener. `IEx.App` does not start `MixListener` (verified: `app.ex:15` children are `IEx.Config`, `IEx.Broker`, `IEx.Pry` only), so the process is started by Mix (`iex -S mix` / equivalent) — production releases without Mix never start it. Developer/shared-build-host exposure only.
- `IEx.Config.auto_reload?()` is a plain boolean app-env read (`config.ex:100-102`, `Application.fetch_env!(:iex, :auto_reload)`; validated boolean at `config.ex:208`) — verified by reading the neighbour. Under `auto_reload? == false` the immediate purge path (lines 54-55) is skipped and modules are merely accumulated into `to_purge` for a later `purge/0`; the destructive `:code.purge`/`:code.delete` still eventually runs, just deferred. My execution set it to `true` to exercise the immediate path.

## Did I verify this?
- **Destructive sink (module unload):** yes — executed under OTP 29; `is_loaded before {:file,[]}` → `after false`, call → `UndefinedFunctionError`.
- **Destructive sink (process kill via double-purge):** yes — executed; worker `true → true → false`.
- **Malformed-diff crash:** yes — executed; `MatchError`/`{:badmatch, %{}}` at line 51, GenServer terminated.
- **os_pid gate bypassable / reaches line 51:** yes — executed (`"0"` passed the check).
- **No atom-creation sink in this file:** yes — read all 76 lines; none present.
- **`auto_reload?` semantics:** yes — read `config.ex:100-102,208`.
- **MixListener not supervised by IEx.App / no in-tree start_link:** yes — read `app.ex`, grepped `lib/iex`.
- **Transport is unauthenticated / world-listable /tmp / `binary_to_term` without `[:safe]` / ListenerSupervisor restart intensity:** **no** — `Mix.Sync.PubSub`/`Mix.PubSub`/`Mix.PubSub.ListenerSupervisor` are not in this checkout; could not Read or re-run. These remain asserted.

## Dropped in re-review
None. Both prior findings are retained. The change from the first pass is a **downgrade of the reachability half of `CSCAN-mix_listener-untrusted-code-purge` and the restart-escalation half of `CSCAN-mix_listener-malformed-diff-crash` from `verified` to `asserted`**, because the transport/supervisor source is absent from this checkout and the prior end-to-end forgery evidence is not reproducible here. The in-file mechanics of both findings were re-verified by execution and their confidence is unchanged (medium / high respectively).

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Atom-table exhaustion in this file** — `changed`/`removed` reach only `:code.purge`/`:code.delete`, which no-op on unknown atoms and create none. The new-atom creation on the wire happens in `Mix.Sync.PubSub.decode_data/1` (it calls `non_executable_binary_to_term/1` without `[:safe]`) — a real issue, but in the transport, not this file.
