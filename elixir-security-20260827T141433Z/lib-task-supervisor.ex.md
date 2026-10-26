# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/task/supervisor.ex
<!-- sha256: 46f79e40f789d15ed5dc857d2b215c7c016510b691478d7e0699c6866db7eaf4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:47:13Z_

Confirmed: the out-of-contract `shutdown` value is quietly accepted, the shared supervisor stays alive, and an unrelated concurrent task completes normally — **no cross-caller blast radius**. This settles the last plausible concern.

I have read the entire file, confirmed the absence of every dangerous sink class by exhaustive grep over the single 656-line file, and verified the by-design behaviors by running them. Here is the report.

---

## Summary

`Task.Supervisor` is clean standard-library glue whose only "code execution" is running the caller-supplied function/MFA it exists to run; no atom-creation, path, command, deserialization, crypto, or Unicode sinks are present, and no untrusted-data-to-sink path is reachable through its documented public API.

## Security

`No findings`

The module's public API (`async/3,5`, `async_nolink/3,5`, `async_stream[_nolink]/4,6`, `start_child/3,5`, `terminate_child/2`, `children/1`, `start_link/1`) is a thin wrapper over `DynamicSupervisor` and an internal `Task.Supervised` message protocol. The functions `apply` a function/MFA the caller passes in — that is the module's documented purpose, equivalent to `apply/3`. There is no path where data *this file* treats as inert becomes code, no conversion of external bytes to atoms/terms/paths, and no shell/network/filesystem interaction. A grep over the whole file for `to_atom`, `Module.concat`, `Code.*`, `eval`, `binary_to_term`, `:os.cmd`, `System.cmd/shell`, `Path.*`, `File.*`, `Regex`, `EEx`, `:zip`, `:erl_tar`, and `Base.decode` returns only a documentation mention of the `:zip_input_on_exit` option (line 363).

### Ruled out

- **Code execution via `async`/`start_child`/`async_stream` (fun & `{module, fun, args}`) — lines 191-193, 213-214, 300-301, 319-320, 389-392, 540-544, 555-568, 610-628.** Running the caller-supplied callable is the documented contract, exactly like `apply/3`. Weaponizing it requires the *caller* to forward attacker-controlled code or a module/function reference into the sink, which is caller misuse explicitly excluded by scope ("a finding that requires the victim to run … on attacker input themselves is not a finding about this repository"). *Basis: verified* — ran `async`/`async_stream`/`start_child`; they execute the caller-provided callable and nothing else.
- **`async_stream` enumerable elements / `args` as an injection vector — lines 389-392, 434-436.** Elements are prepended to `args` and passed as ordinary function arguments, never evaluated as code. Forwarding an untrusted *collection* is safe; it is data. *Basis: verified* — `async_stream(:sup, [1,2,3], fn x -> x*10 end)` returned `[ok: 10, ok: 20, ok: 30]`, i.e. values are consumed as data.
- **Atom-table exhaustion (CWE-770).** No `String.to_atom`/`binary_to_atom`/`Module.concat`/`list_to_atom` on any value. `child_spec/1` (lines 104-109) only *matches* on an already-atom `:name`; it never creates atoms. *Basis: verified* — exhaustive grep of the single file.
- **Shared-state poisoning via `Process.put(__MODULE__, arg)` — line 169.** Writes to the supervisor's *own* process dictionary, once at init, with `{restart, shutdown}` derived from `start_link/1` options (operator/developer config, set at boot). It is not keyed or valued by any per-call untrusted data and is not cross-caller reachable. *Basis: asserted* (single writer at init; value flows only from `start_link` options at lines 151-164).
- **`shutdown`-validation asymmetry — `build_stream` validates (lines 633-635) but `async`/`start_child` do not.** Passing an out-of-contract `:shutdown` (e.g. `:bogus`) is a documented-type-spec violation (excluded by scope), and its blast radius is nil: the value is stored in the child spec and only consulted at child-termination time. *Basis: verified* — `async(sup, fn -> :x end, shutdown: :bogus)` returned a normal `%Task{}`, the supervisor stayed alive (`Process.alive?/1 → true`), and an unrelated concurrent task completed (`:victim_ok`). No supervisor-wide DoS.
- **`child_spec/1` `CaseClauseError` on a non-`{atom | {:global,_} | {:via,_,_}}` name — lines 104-109.** The `:name` comes from the developer's child specification, not an untrusted boundary; a crash here is a startup misconfiguration in the operator's own supervision tree.
- **`GenServer.call(supervisor, …, :infinity)` — line 590.** Unbounded wait is a deliberate design choice so task starts aren't spuriously timed out; it is not amplifiable by untrusted *data* passing through the public API.
- **Unbounded task creation / `children/1` OOM — lines 508-511, 610-628.** Bounded by the documented `:max_children` option (`async` raises on `:max_children`), and the `children/1` OOM caveat is called out explicitly in its own `@doc`. These are documented, operator-controlled capacity decisions, not hidden defects.
- **`:link`-after-start race — lines 616, 647.** The spawned `Task.Supervised` process blocks awaiting its instruction message before doing any work, so it cannot terminate between `{:ok, pid}` and `Process.link/1`; there is no missed-exit window. *Basis: asserted* (protocol reading — the task runs nothing until it receives `{owner, alias, alias, callers, mfa}`).

## Risks / caveats

- **Cross-file trust boundary not re-audited here.** Correctness of stream-option validation and the actual task lifecycle lives in `Task.Supervised` (`validate_stream_options/1`, `stream/7`) and `DynamicSupervisor`; per instructions I reviewed only this file and treated those as trusted collaborators. A defect in `Task.Supervised.validate_stream_options` (e.g. accepting a hostile `:max_concurrency`) would surface through this file's `async_stream*` API but is out of this file's scope.
- **Contract-violation behavior is only spot-checked.** I verified that a bogus `:shutdown` does not harm the shared supervisor, but did not fuzz every out-of-type option value (`:restart`, negative timeouts, malformed `supervisor` refs); these are excluded as type-spec violations, and no cross-caller amplification was observed in the case tested.
- **Residual risk is caller-side, by design.** The one real way this module leads to code execution is a caller forwarding untrusted code/module references into `async`/`start_child`/`async_stream`. That is inherent to a task-spawning API and cannot be defended within this file; it is an application-level concern, not a defect here.

