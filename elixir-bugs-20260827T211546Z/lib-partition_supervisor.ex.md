# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/partition_supervisor.ex
<!-- sha256: d6dff430ebf8eb94af6e8f91870555cbe44379be410bac03b1e31f46c994841e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:34:00Z_

Confirmed a concrete contract/doc violation. Let me verify one more angle — whether `:restarting` (which the doc claims) versus `:undefined` distinction, and confirm the `count_children` `:specs` doc claim divergence is real (both shown above: `specs: 4` while `partitions()` is 2).

Both are reproduced. Let me write the report.

## Summary

`PartitionSupervisor` routes and resizes correctly, but after a documented `resize!/2` down-scale the reporting functions `which_children/1` and `count_children/1` return values that contradict their own `@spec`/`@doc` (terminated partitions surface as `:undefined`, and `:specs` no longer equals the partition count).

## Bugfixes

#### `CSCAN-partition_supervisor-which-children-undefined-shape`
- **Finding ID** — `CSCAN-partition_supervisor-which-children-undefined-shape`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `partition_supervisor.ex:471-477` (`which_children/1`, `@spec` at 471-474, `@doc` `child` bullet at 461-462)
- **Issue** — The `@doc` states each tuple's second element is "the PID of the corresponding child process or the atom `:restarting`", and the `@spec` types it as `pid | :restarting`. After a `resize!/2` down-scale, the terminated partitions are reported by the underlying supervisor as `:undefined`, a third shape absent from both the doc and the spec.
- **Class** — Contract violation (also Documentation mismatch).
- **Impact** — A caller relying on the published contract — e.g. `case child do :restarting -> …; pid -> use(pid) end`, or any Dialyzer analysis narrowed to `pid | :restarting` — will silently treat the `:undefined` atom as a live PID (leading to a downstream crash when it is used as one) or be flagged by Dialyzer. `resize!/2` (public since 1.18) plus `which_children/1` (public since 1.14) reach this purely through documented API; no misuse required.
- **Evidence** — `verified`. Running `PartitionSupervisor.start_link(child_spec: {Collector, []}, name: PSW, partitions: 4)` then `PartitionSupervisor.resize!(PSW, 2)` then `which_children(PSW)` produced:
  ```
  [{3, :undefined, :worker, [Collector]},
   {2, :undefined, :worker, [Collector]},
   {1, #PID<0.104.0>, :worker, [Collector]},
   {0, #PID<0.103.0>, :worker, [Collector]}]
  ```
  The `:undefined` value matches neither the doc text nor the `pid | :restarting` spec.
- **Suggested fix** — Add `:undefined` to both the `@doc` (document that resized-away/terminated partitions report `:undefined`) and the `@spec` (`pid | :restarting | :undefined`).

#### `CSCAN-partition_supervisor-count-specs-stale-after-resize`
- **Finding ID** — `CSCAN-partition_supervisor-count-specs-stale-after-resize`
- **Severity** — Low
- **Confidence** — high
- **Location** — `partition_supervisor.ex:479-505` (`count_children/1`, `:specs` bullet at 484)
- **Issue** — The `@doc` defines `:specs` as "the number of partitions (children processes)". After a `resize!/2` down-scale, terminated partitions remain as child specs, so `:specs` stays at the original maximum while the actual partition count (`partitions/1`) is lower; the two diverge despite the doc equating them.
- **Class** — Documentation mismatch.
- **Impact** — A caller using `count_children(name).specs` as "how many partitions do I currently have" (as the doc invites) gets the original count, not the resized one, and must instead call `partitions/1`. Wrong capacity/health metrics rather than a crash.
- **Evidence** — `verified`. Same scenario as above: after `resize!(PSW, 2)`, `count_children(PSW)` returned `%{active: 2, workers: 4, supervisors: 0, specs: 4}` while `partitions(PSW)` returned `2` — `:specs` (4) ≠ number of partitions (2).
- **Suggested fix** — Reword the `:specs` bullet to "the total number of partition child specs (unchanged by `resize!/2`)", or direct readers to `partitions/1` for the current active partition count.

## Risks / caveats

- **Zero-partition routing raises rather than returning `:undefined`.** `resize!(name, 0)` is documented as allowed; afterward any `{:via, PartitionSupervisor, {name, key}}` routing (via `whereis_name/1`) raises `ArgumentError "… has zero partitions"` in the *caller* process (verified), instead of the `:undefined`/`:noproc` a via callback conventionally yields. This looks like a deliberate design choice (there is genuinely nowhere to route), so I did not file it as a bug, but callers draining to 0 partitions should expect a raise, not a clean `nil`.
- **Stale ETS PID during the crash→restart window** is inherent to the ETS-backed via pattern (a routed message can hit a just-crashed PID before `start_child` re-registers); this is the documented trade-off of the approach, not a defect in this file.
- I reviewed every exported/`@doc false`-but-via-reachable function (`start_link`, `resize!`, `partitions`, `which_children`, `count_children`, `stop`, `whereis_name`, `send`, `child_spec`). The routing arithmetic (`rem(abs(key), partitions)` / `phash2`), resize range bounds (`current..(partitions-1)` up, `partitions..(current-1)` down, `0..max//1` guard), and ETS-count ordering (lower-before-terminate / restart-before-raise) were all exercised and behaved correctly on valid input.

### Ruled out
- **Off-by-one in resize ranges** — `verified`: down-scale 4→2 terminates exactly ids 2,3 and up-scale 2→4 restarts exactly ids 2,3; routing after each resize hits only live partitions (test output above). No off-by-one.
- **Wrong/stale routing after resize** — `verified`: post-resize `whereis_name` returned live PIDs for all keys, and re-scaled partitions received fresh PIDs (`0.110`,`0.111`), confirming `start_child` re-registration overwrites stale ETS rows.
- **`restart_child`/`terminate_child` return-shape gaps in `resize!`** — `asserted`→reasoned against OTP's documented return set: `restart_child` yields `{:ok,_}|{:ok,_,_}|{:error,_}` (all three matched, error raises) and `terminate_child` yields `:ok|{:error,_}` (both matched); no unhandled shape.
- **`start_link` required-option handling** — `verified`: omitting `:name` or `:child_spec`, or passing non-positive `:partitions`/bad `:with_arguments`, raises the documented `ArgumentError` (consistent with the `@doc`), so these are contract-conformant, not bugs.

