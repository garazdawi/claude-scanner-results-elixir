# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit.ex
<!-- sha256: 8c3b3e6b889c25d2a342d7688b5f8f2711b056deeb4eceda73d4a441c3d129a3 -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit.ex
<!-- sha256: 8c3b3e6b889c25d2a342d7688b5f8f2711b056deeb4eceda73d4a441c3d129a3 -->

_Re-analysed on 2026-08-27_

## Summary

`ex_unit.ex` is ExUnit's top-level `Application`/configuration/lifecycle module (suite `start`/`run`/`configure`/`configuration`, pluralization rules, and the test-supervisor accessor); it is pure orchestration glue whose only inputs are developer-controlled configuration and caller-supplied `[module()]` lists. I found no security defect reachable through its documented public API. This re-review replaces the two first-pass hedges ("I assumed the documented contracts hold", "I assume") with executed evidence.

## Security

No findings.

### Ruled out

- **`run/1` dynamic module dispatch (`module.__ex_unit__(:config)`, lines 478–484)** — sink-shaped (dynamic `apply` + `Code.ensure_loaded?/1` code loading), but gated by `Code.ensure_loaded?(module) and function_exported?(module, :__ex_unit__, 1)`, and the modules come from the documented `[module()]` argument supplied by the trusted caller, not from network/disk/archive input. `verified` (RAN, OTP 29): for a loaded non-ExUnit module `Code.ensure_loaded?(Enum) and function_exported?(Enum, :__ex_unit__, 1)` → `false` (guard fires → `ArgumentError`, no dispatch); for a never-loaded atom `Code.ensure_loaded?(:"Elixir.NoSuchModuleXYZ")` → `false` and `function_exported?/3` → `false` (guard fires). Axes examined: loaded-non-case, unloaded, dispatch-reachability | Not examined: none.
- **`run/1` caller-contract violation (non-atom in `additional_modules`)** — **Upgraded scope from hedged assumption in first pass.** The first pass wrote "I assumed … `run/1` receives real module atoms." Rather than assume, I ran it: `Code.ensure_loaded?(123)` raises `FunctionClauseError` (the BIF requires an atom). So a caller passing a non-atom does not reach dispatch — the `for` body crashes with `FunctionClauseError` before any `apply`. This is a self-inflicted crash from violating the `[module()]` spec (out of scope per review rules) with no privilege crossing and no path to arbitrary-module dispatch. `verified` (RAN). Confidence: high (not a finding). Axes examined: non-atom (integer) input | Not examined: none.
- **`configure/1` → `Application.put_env(:ex_unit, k, v)` for arbitrary `k`/`v` (lines 421–425)** — global VM state write, but writing `:ex_unit` app env *is* the documented purpose of `configure/1`; there is no second tenant to poison (the developer configures their own suite). Not a privilege crossing. `configure/1`/`start/1` receiving a non-keyword shape is guarded by `when is_list(options)` (and `Enum.each` over a non-`{k,v}` element would raise a self-inflicted `MatchError` in the caller's own process) — a caller-spec violation, out of scope, no security impact. Axes examined: arbitrary key/value, non-keyword input | Not examined: none.
- **`after_suite/1` storing a caller function later invoked (lines 531–534); `:formatters` modules dispatched later** — callables/modules are supplied by the developer registering their own callbacks/formatters; no externally influenced type or module name. `is_function/1` guard present. Axes examined: callable provenance | Not examined: none.
- **`TimeoutError.message/1` (line 242) and `plural_rule/1` (line 448) string interpolation** — interpolate an internal `type`/`timeout` and a caller binary `word` into plain human-readable strings; no downstream interpreter (no shell, query, regex, EEx, or format-string sink), so no injection.
- **`fetch_test_supervisor/0` check-then-act (lines 546–558)** — `get_supervisor`→`start_link`→`put_supervisor` is keyed on `self()`; a process cannot race itself, and it is documented to run in the test process, so no TOCTOU.
- **Manifest `binary_to_term` and `Filters` atom-table growth** — the folder threat model's two genuine adversarial surfaces are implemented in `FailuresManifest` and `Filters`, not in this file; `ex_unit.ex` never touches the manifest bytes or CLI filter tokens.
- **`put_slowest/1` term-ordering quirk (`opts[:slowest] > 0`, line 613)** — `nil > 0` is `true` in Elixir term order — `verified` (RAN: `nil > 0` → `true`) — so an unset `:slowest` would force trace mode; but `mix.exs` sets `slowest: 0`/`slowest_modules: 0` as app-env defaults, so the `nil` case is unreachable via the public API, and forcing trace mode is a robustness/behavior issue with no security impact. Axes examined: nil-vs-integer ordering | Not examined: none.

## Risks / caveats

- Scope is this single file. The dynamic dispatch and `configure` writes are safe *here* because inputs are trusted at this boundary; the security-relevant consumers of the app env this file populates (formatter dispatch in `CLIFormatter`/`Formatter`, filter evaluation in `Filters`, manifest read/write in `FailuresManifest`) live in sibling files and are out of scope for this file's review — they are not re-adjudicated here.
- Caller-contract violations resolved by execution rather than assumption: passing a non-atom to `run/1` raises `FunctionClauseError` before dispatch (RAN), and passing a non-keyword to `configure/1` is blocked by `is_list/1` or raises a self-inflicted `MatchError` in the caller's process — both are out-of-scope spec violations with no security impact, not silent misbehavior.
- No compile-time execution risk exists in this file: the `defstruct`/`defexception`/`use Application` expansions are pure and read no filesystem or network input at build time.

## Did I verify this?

- `run/1` dispatch guard fires for non-ExUnit loaded and unloaded modules — **verified (RAN, OTP 29).**
- `run/1` non-atom input crashes before dispatch (`Code.ensure_loaded?(123)` → `FunctionClauseError`) — **verified (RAN).**
- `nil > 0 == true` term ordering, making the `put_slowest` `nil` branch a trace-forcer if reachable — **verified (RAN);** unreachable via public API because `mix.exs` supplies `slowest: 0` default.
- `configure/1` is `is_list/1`-guarded; sibling consumers of `:ex_unit` app env — **verified guard by reading source; consumers out of scope (not re-adjudicated).**

### Dropped in re-review

None. Every prior ruled-out entry is preserved; two were strengthened with executed evidence and the first-pass "I assumed"/"I assume" hedges were removed by running the corresponding paths.
