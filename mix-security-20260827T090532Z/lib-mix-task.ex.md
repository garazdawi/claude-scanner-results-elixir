# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/task.ex
<!-- sha256: 8b468b070be109fafecfd132d0cf2e3b71a214e18b4035c25030dd0ef13dee2b -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/task.ex
<!-- sha256: 8b468b070be109fafecfd132d0cf2e3b71a214e18b4035c25030dd0ef13dee2b -->

_Re-analysed on 2026-08-27_

## Summary

`Mix.Task` is defensively clean. Its security-relevant sinks are all **unbounded-atom (CWE-770)** vectors: the `String.to_unsafe_atom/1` alias-name lookups, and — verified this pass by reading the neighbour `Mix.Utils` — the `Module.concat/2`-based module resolution behind `get/1`/`fetch/1`. Both turn a task-*name* binary into a permanent atom before establishing that the name corresponds to any real alias or module. A separate narrow crash (`ArgumentError` out of `load_tasks/1`) is reachable only under the unsupported latin1 filename mode; the latin1 axis is now tried and reproduced.

Three hedges from the first pass are resolved below: two by reading the neighbour module and executing the sink (`command_to_module` → `Module.concat`), one by trying the previously-unexamined latin1 filename axis. No prior finding is dropped.

## Security

#### `CSCAN-task-alias-lookup-atom-leak`
- **Finding ID** — `CSCAN-task-alias-lookup-atom-leak`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `lib/mix/task.ex:312` (`alias?/1`); same defect at `:422` (`do_run/3`) and `:444` (`recur_umbrella_children_alias/3`)
- **Class** — Atom creation (CWE-770)
- **Impact** — Each site turns a task-*name* binary into a brand-new permanent atom via `String.to_unsafe_atom/1` purely to index the `:aliases` keyword list, *before* establishing the name corresponds to any real alias. Atoms are never garbage-collected and the table is capped (default `1_048_576`). A caller forwarding externally-influenced names into the documented public `Mix.Task.alias?/1` (or into `run/2`, which reaches line 422) leaks one permanent atom per distinct name and can eventually abort the VM with `system_limit`. The atomization is gratuitous for the alias check: alias keys in `Mix.Project.config()[:aliases]` are already atoms once config compiles, so an unknown name provably cannot be an alias key and never needs to be created.
- **Evidence** — `verified`. Driving `Mix.Task.alias?/1` with 5000 fresh strings (first-pass harness, still valid):
```
atoms before=19243 after=24307 delta=5064
fresh name 'cscan_bogus_task_4242' now exists as atom? true
alias?/1 on a bogus name returns: false
```
Every non-alias name became a permanent atom while `alias?/1` returned `false`. A compiled module carrying `"my.alias":` config makes `String.to_existing_atom("my.alias")` succeed, confirming the checked form is viable.
- **New note this pass** — On the `run/2` → `do_run/3` path specifically, the line-422 `to_unsafe_atom` leak is *partially redundant* with the downstream `get(task)` leak (see `CSCAN-task-get-module-concat-atom-leak` below): even if line 422 were fixed to `to_existing_atom`, an unknown name still gets atomized by `maybe_load_or_compile_task → get`. The **standalone `alias?/1` public entry (line 312) is the clean, fully-demonstrated leak** because it never calls `get/1`.
- **Suggested fix** — Use `String.to_existing_atom/1` for the alias-key lookups at `:312`/`:422`/`:444`, treating `ArgumentError` as "not an alias". Note the fix is only fully effective when combined with the get-path fix below.

#### `CSCAN-task-get-module-concat-atom-leak` — Upgraded from hedged assumption in first pass
- **Finding ID** — `CSCAN-task-get-module-concat-atom-leak`
- **Severity** — Low
- **Confidence** — low
- **Location** — `lib/mix/task.ex:357` (`fetch/1`, reached by `get/1:326`, `get!/1:343`, `do_run/3:431`, `run_alias/6:571`); sink itself is `Mix.Utils.command_to_module/2` at `lib/mix/utils.ex:266-269`
- **Class** — Atom creation (CWE-770)
- **Impact** — The first pass asserted the module-resolution path "legitimately needs the atom to load the module" and marked its atom-safety "out of scope for this file." Reading the neighbour and executing it **disproves the "legitimately/bounded" premise**: `command_to_module` does `Module.concat(at, command_to_module_name(command))`, and `Module.concat/2` unconditionally creates a permanent atom **even when the target module does not exist and never loads**. So every distinct task-name string flowing through `fetch/1` (i.e. `get/1`, `get!/1`, and the `run/2` fall-through at line 431) leaks one permanent atom regardless of whether a task is found — the same unbounded-atom vector as the alias sites, on the far more heavily-used resolution path.
- **Evidence** — `verified` (executed). 3000 fresh non-existent names through `Module.concat(Mix.Tasks, ...)`:
```
delta=3001            # one permanent atom per distinct name
atom_exists=true      # String.to_existing_atom("Elixir.Mix.Tasks.Cscan_nope_2000") succeeds afterward
```
The module never existed, yet its atom is now permanent.
- **Confidence rationale** — Low because the atom-creating sink (`Module.concat`) lives in `Mix.Utils`, not this file; this file only supplies the call sites. The finding is recorded here so a later cross-file pass re-examines the `command_to_module` sink rather than inheriting the first pass's "out of scope / legitimately bounded" dismissal.
- **Suggested fix** — Cross-file: `command_to_module` could resolve via `String.to_existing_atom` + `Code.ensure_loaded` and treat `ArgumentError` as `{:error, :nofile}`, since a task module that is loadable already has its atom interned by the code server.

#### `CSCAN-task-load-tasks-latin1-crash` — Upgraded from hedged ruled-out entry in first pass
- **Finding ID** — `CSCAN-task-load-tasks-latin1-crash`
- **Severity** — Low
- **Confidence** — low
- **Location** — `lib/mix/task.ex:191` (`task_from_path/1`), propagating out of `load_tasks/1:165` / `load_all/0:159`
- **Class** — Uncaught exception / robustness (DoS-flavored)
- **Impact** — Under the **latin1 native name encoding** (`+fnl` / unset `+fnu`, which Elixir warns against at startup), `File.ls` returns raw-byte filenames instead of dropping non-UTF-8 names. A file named `Elixir.Mix.Tasks.<invalid-utf8>.beam` on the code path then matches the binary pattern at line 190, and `String.to_unsafe_atom("Elixir.Mix.Tasks." <> <<0xff,0xfe>>)` raises `ArgumentError: invalid UTF8 encoding` (it is `binary_to_atom(_, :utf8)`). Nothing in `task_from_path/1` or `load_tasks/1` catches it, so `Mix.Task.load_all/0` crashes. A malicious `.ez` archive can carry an arbitrarily-named `.beam` extracted onto the code path, supplying the trigger file.
- **Evidence** — `verified` (latin1 axis now tried, not merely asserted). With `:file.native_name_encoding() == :latin1`:
```
File.ls: {:ok, ["Elixir.Mix.Tasks.\xff\xfe.beam"]}      # NOT dropped under latin1
:file.list_dir: {:ok, [[...,46,255,254,46,98,101,97,109]]}
# feeding the matched rest to the exact sink:
** (ArgumentError) 1st argument: invalid UTF8 encoding
    :erlang.binary_to_atom(<<...,255,254>>)   # to_unsafe_atom raises -> load_tasks crashes
```
Contrast: under supported `+fnu`, `File.ls` silently drops the same name (first-pass evidence), so no crash — hence the tight precondition.
- **Confidence rationale** — Low: requires the unsupported latin1 config **and** a planted `.beam` on the (already operator-trusted, post-compile) code path. Recorded rather than left ruled-out because trying the axis produced an executed crash, so the axis is no longer "not examined."
- **Suggested fix** — Wrap the `to_unsafe_atom` in `task_from_path/1` in `try/rescue ArgumentError -> nil`, so a malformed filename is skipped rather than aborting task discovery.

### Ruled out

- **`task_from_path/1:191` — too-short filename / negative binary-size** — `Axes examined: byte length (negative `part`), UTF-8 validity of `rest`, native name encoding (utf8 + latin1) | Not examined: none`. Tried directly: `byte_size("x.beam") - 17 - 5` is negative, and `<<..., rest::binary-size(-12), ...>>` simply **fails to match** (returns `nil`), no raise — reproduced (`too-short: nil`). The invalid-UTF-8 sub-case is *not* ruled out under latin1 — split out into `CSCAN-task-load-tasks-latin1-crash` above.
- **`module.run(args):502` dynamic dispatch** — `Axes examined: module provenance (task?/1 + ensure_task?/1 gate), name prefix, export of run/1 | Not examined: none needed`. `module` is gated by `task?/1`+`ensure_task?/1` (name must start `Elixir.Mix.Tasks.`, be loaded, export `run/1`). Executing a resolved task module — including one shipped by a dependency — is Mix's documented design (an installed archive runs arbitrary code by design), not a boundary crossing.
- **`run_requirements/1:516` / alias recursion (`OptionParser.split`)** — `@requirements` and `:aliases` strings are developer/project-config controlled (config executes as code = operator RCE by design). `OptionParser.split` only tokenizes (no shell). `Mix.TasksServer` idempotency guards bound re-entrancy. Not an attacker-controlled surface.
- **`Mix.TasksServer`/`Mix.State` writes** — keyed by the task-name string; an actor who can choose task names already controls task execution, so poisoning that state grants nothing new. (`Mix.TasksServer`'s `:public` ETS is co-tenant-poisonable but co-resident code is operator-trusted — folder threat model.)
- **`to_unsafe_atom` `ArgumentError` on an invalid-UTF-8 CLI task name** — self-inflicted (operator supplies argv); not a dependency/archive/network boundary. (Distinct from the latin1 *filename* path, which is attacker-influenceable via archive contents — that is the upgraded finding.)

## Did I verify this?

- **Main atom leak (`alias?/1`)** — Verified, executed. `delta=5064` permanent atoms from 5000 bogus names; `alias?/1` returns `false` throughout. Checked form (`to_existing_atom`) confirmed viable.
- **Hedge #2 — "get/1 → command_to_module atom-safety out of scope / legitimately bounded"** — **Resolved by reading the neighbour and executing it.** `Mix.Utils.command_to_module` (`utils.ex:266`) uses `Module.concat`, which I ran: `delta=3001` permanent atoms for 3000 never-existent module names; `to_existing_atom` on one afterward succeeds. The "legitimately bounded" premise is false — upgraded to `CSCAN-task-get-module-concat-atom-leak` (low confidence, cross-file sink).
- **Hedge #3 — "did not fully characterize `task_from_path` under latin1 (+fnl)"** — **Axis tried, not asserted.** With `:file.native_name_encoding() == :latin1`, `File.ls` returns the raw-byte name (reproduced), the line-190 pattern matches, and `String.to_unsafe_atom` raises `ArgumentError` (reproduced). Upgraded to `CSCAN-task-load-tasks-latin1-crash` (low confidence, tight preconditions).
- **Hedge #1 — cross-file high-cardinality reachability of untrusted names into `run/2`/`alias?/1`** — Still not exhaustively provable without auditing every Mix task; this bounds *severity/confidence* (Low/medium), not correctness. It is a reachability caveat, not a ruled-out axis, so it remains a caveat rather than being upgraded or cleared.

## Risks / caveats

- The two atom-leak findings share a root cause (permanent atoms minted from task-name strings before validation). Practical exploitability depends on a caller forwarding high-cardinality untrusted names; within this file the fully-demonstrated public entry is `alias?/1`. Hence Low severity throughout.
- `CSCAN-task-get-module-concat-atom-leak`'s sink is in `Mix.Utils`; this report anchors it at the call sites so a `utils.ex` pass does not inherit the prior "out of scope" dismissal.

### Dropped in re-review

None. Every first-pass finding is preserved; the first-pass "Ruled out" latin1 sub-case and the "out-of-scope get path" caveat were *upgraded* (not dropped) after executed evidence, per the prefer-safe-direction rule.
