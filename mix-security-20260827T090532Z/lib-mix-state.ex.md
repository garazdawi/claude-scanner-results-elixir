# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/state.ex
<!-- sha256: 6fc72b6cf2a4cb813714aa50dc6c1086c3b6418d4b0d899a740d31881b419747 -->

_Analysed on 2026-08-27T10:08:33Z_

Confirmed. The analysis is complete. Key findings from reproduction:

1. **`builtin_apps` snapshots `:code.get_path()` at GenServer-init time** and caches the computed map — late-added dependency ebin dirs (`cscan_late_dep`) are never picked up. The snapshot contains only standard OTP/Elixir/Mix apps (`:erl_interface, :odbc, :megaco, ...`). So `List.to_unsafe_atom` in `app_from_code_path` never sees dependency-controlled directory names.
2. `String.to_unsafe_atom`/`List.to_unsafe_atom` are real functions that create atoms unconditionally (same semantics as `to_atom`); `to_existing_atom` is the safe form.
3. Atom length caps at 255; a 256-char component raises `SystemLimitError`, but filesystem path components can't exceed 255 bytes and the snapshot is standard-lib anyway.

Here is my report.

## Summary
`Mix.State` is an internal (`@moduledoc false`) GenServer/ETS/persistent-term state store whose only external-input touchpoints — atom creation from `MIX_ENV`/`MIX_TARGET` and from code-path directory names — are bounded and operate on operator- or standard-library-controlled data, so no attacker-controlled path reaches a dangerous sink.

## Security

No findings.

### Ruled out

- **`from_env/2` → `String.to_unsafe_atom(value)` (state.ex:91) — atom creation from env vars.** Matches the CWE-770 pattern (unconditional atom creation from external bytes), but the source is `MIX_ENV`/`MIX_TARGET`, which are operator/CI-supplied environment variables (per the folder threat model, operator config, not a dependency/archive/network boundary), and `init/1` calls it exactly twice per VM — at most two atoms per process lifetime, not exhaustion. Unconditional creation is also intentional here: developers define custom envs (`MIX_ENV=integration`), so `to_existing_atom` would break legitimate use. Not reachable with attacker-controlled, unbounded data.
- **`app_from_code_path/1` → `List.to_unsafe_atom(app)` (state.ex:115) — atom creation from code-path directory names.** `init/1` snapshots `:code.get_path()` at GenServer-start time (`state = %{builtin_apps: :code.get_path()}`), and `handle_call(:builtin_apps, …)` computes the app→path map from that frozen snapshot once and caches it. Verified that a dependency ebin dir added *after* Mix boot is never included: `elixir -e 'Application.ensure_all_started(:mix); a1=Mix.State.builtin_apps(); :code.add_pathz(~c"/tmp/deps/cscan_late_dep-1.0.0/ebin"); a2=Mix.State.builtin_apps()'` → `late dep in map (1st)? false`, `(2nd)? false`, `map identical (cached)? true`, sample keys `[:erl_interface, :odbc, :megaco, :debugger, :parsetools, :eex]`. Mix adds dependency load paths only later, during task execution, so dependency directory names never flow into `List.to_unsafe_atom`; the atoms created are standard OTP/Elixir app names that already exist. Bounded and trusted.
- **`List.to_unsafe_atom` `SystemLimitError` on a >255-char component.** Verified 255 chars succeeds and 256 raises `SystemLimitError`, but (a) a single filesystem path component cannot exceed 255 bytes on normal filesystems, and (b) the code path processed is the init-time standard-library snapshot, not attacker dirs. Unreachable in practice; would at worst crash `builtin_apps` self-inflictedly.
- **`:public` ETS table (state.ex:71) and `put/2`,`update/2`,`get/2`,`fetch/1`.** The table is `:public`, so any co-resident process can read/write it, but per the threat model such code is already operator-trusted local infrastructure; there is no remote-content path to it. Keys/values stored (`:shell`, `:env`, `:target`, `:scm`) are Mix-internal, not externally derived.
- **`update/2` non-atomic read-modify-write (state.ex:42).** `:ets.lookup_element` then `:ets.insert` is a lost-update race on the `:public` table, but the racers are trusted Mix processes, not a trust boundary; not security-relevant.
- **`write_cache`/`read_cache`/persistent_term (state.ex:48–65).** `:persistent_term.put` triggers a global GC and keys are `{__MODULE__, key}`, but keys/values are Mix-internal, not attacker-derived; no cross-boundary poisoning from this file's API.

## Risks / caveats
- `Mix.State` is `@moduledoc false`; its "public API" is what Mix itself calls. I traced reachability of the two atom sinks from Mix startup (`init/1`) and the `builtin_apps` request path, and reproduced the frozen-snapshot behavior against the built runtime (Elixir 1.21.0-dev / OTP 29) — dependency directory names do not reach `app_from_code_path`.
- I did not audit the *callers* of `put/2`/`update/2`/`write_cache/2` in other files (out of review scope); if some other module were to forward attacker-controlled bytes as ETS/persistent-term *keys* in an unbounded loop, that would be a finding in that file, not here.
- `String.to_unsafe_atom`/`List.to_unsafe_atom` are the OTP-29-era explicit names for unconditional atom creation (semantically identical to `to_atom`); the safe alternative for both sinks is `to_existing_atom`, which is inapplicable here because legitimate new envs/apps must be creatable — reinforcing that these are by-design, not defects.

