# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/dynamic_supervisor.ex
<!-- sha256: 4d58038be32f32fa31d39791460bb28c847115fe52ea269b80ce4f1313be1646 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:38:47Z_

Confirmed. The only match is `Process.put(:"$initial_call", {:supervisor, mod, 1})` (line 597) — a fixed atom key in the supervisor's *own* process dictionary, standard `proc_lib` convention. None of the dangerous-sink families (atom creation from bytes, code eval, command/file/path, deserialisation, crypto, regex, archive) are present. The two genuine sinks — `apply/3` and `mod.init/1` — are the supervisor's defining purpose, fed caller-supplied specs, with no in-module path that turns external file/network bytes into either.

---

# Security review — `dynamic_supervisor.ex`

## Summary
`DynamicSupervisor` is a general-purpose OTP supervisor behaviour whose only sinks (`apply/3`, `mod.init/1`) execute caller-supplied child specifications by design; it parses no external bytes and reaches no dangerous stdlib primitive, so it exposes no security defect through its documented public API.

## Security

`No findings`

### Ruled out

- **`apply(m, f, extra ++ args)` in private `start_child/3` (line 766) — code execution, intended and contained.** The MFA comes from the child specification the caller hands to `start_child/2`; telling a supervisor what to start *is* the API. The value is validated (`validate_start/1` requires `{atom, atom, list}`, line 437) and the call is wrapped in `try/catch` (lines 765–777), so hostile/invalid specs return `{:error, _}` without taking the supervisor down. There is no path inside this module that turns external file/network bytes into that MFA — the trust boundary is the developer writing the spec. *Basis: verified* — `start_child` with an ill-shaped MFA returned `{:error, {:invalid_mfa, …}}`, a raising start returned `{:error, {:undef, …}}`, and a non-pid return normalised to `{:error, #PID<…>}`, with `Process.alive?(sup)` remaining `true` in all three cases.
- **Behaviour dispatch `mod.init(init_arg)` (lines 600, 843) — intended.** `mod` is the module the caller passes to `start_link/3`/module-based supervisors; this is the documented `use DynamicSupervisor` pattern, not dispatch on an externally-influenced type. *Basis: code reading.*
- **Unbounded children / resource consumption.** `max_children` defaults to `:infinity` (line 579) and the module is explicitly designed to hold millions of children; the limit is the caller's configuration choice, not a module defect. The `map_size(children) < max_children` gate (line 742) works because integers sort below the `:infinity` atom in Erlang term order. *Basis: code reading.*
- **`Process.put(:"$initial_call", {:supervisor, mod, 1})` (line 597) — shared-state write.** Fixed atom key, the supervisor's own process dictionary (single writer = itself), value is the caller's module. Standard `proc_lib` labelling; no cross-process poisoning surface. *Basis: verified via named grep — it is the only `Process.put`/`ets`/`persistent_term`/`Application.put_env` site in the file.*
- **`{:start_task, args, restart, shutdown}` handler + `Process.get(Task.Supervisor)` (lines 723–724).** Not part of `DynamicSupervisor`'s documented public API — it is an internal protocol driven by `Task.Supervisor`. On a plain `DynamicSupervisor` the `Process.get` returns `nil` and the tuple match would crash the process, but reaching it requires sending a crafted message to the process, which the ground rules exclude and which presupposes on-node access (a co-tenant BEAM process is not a defended boundary). *Basis: scope — crafted internal message, not a public entry point.*
- **`child_spec/1` name `case` (lines 212–216).** A `:name` outside `atom | {:global, _} | {:via, _, _}` raises `CaseClauseError`, but that violates the documented `:name` type contract and only crashes the local caller during spec construction (no supervisor involved yet). *Basis: contract violation — excluded per ground rules.*
- **Unexpected-message logging (`handle_info/2` line 823, `format_report/1` line 1119).** The format string is the literal `~p` (not `~s`), so a logged message term cannot inject format directives; and reaching it requires the ability to send messages to the process (on-node), outside the threat model. *Basis: code reading — literal `~p` format.*
- **Supervisor/error reports carrying child MFA and exit reason (`report_error/1` line 1079).** Emitted as structured `:logger` reports via `:logger.format_otp_report/1`; values are report fields, not interpolated into any downstream interpreter. *Basis: code reading.*

## Risks / caveats
- The review is scoped to this file. `Supervisor.child_spec/2` (called at line 393) normalises specs upstream and was not audited here; the containment argument for the `apply` sink relies only on this file's `validate_child/*` chain and `try/catch`, both of which are present and were exercised.
- The verified runs used the in-repo Elixir 1.21.0-dev build; behaviour matches the source read. A latin1 locale warning appeared but is unrelated to the paths tested.
- "No findings" here is a claim about the **documented public API** (`child_spec/1`, `start_link/1,3`, `start_child/2`, `terminate_child/2`, `which_children/1`, `count_children/1`, `stop/2,3`, `init/1`). Faults reachable only by delivering crafted raw messages to the supervisor process (e.g. the `:start_task` path) are deliberately out of scope and were not treated as findings.

