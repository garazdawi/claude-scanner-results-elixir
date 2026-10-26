# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_sup.erl
<!-- sha256: 3e7685b4be220e44e6142cc3bd1167563bbdf692374ab07bbec85412a88f48fb -->

_Analysed on 2026-08-27T17:27:16Z_

Confirmed the complete trust picture. `elixir_sup:start_link/0` is called exactly once, from `elixir:start/2` (the OTP application boot callback at `elixir.erl:107`), with no arguments; `init/1` receives the hardcoded atom `ok`. There is no parameter through which external data can flow into this file. This is pure OTP supervision boilerplate with a fully static child specification.

# Security review — `elixir_sup.erl`

## Summary
`elixir_sup.erl` is a 36-line OTP supervisor whose entire behaviour is to start two singleton workers from a hardcoded, static child specification with no external input on any path, and it contains no security-relevant sinks.

## Security

No findings.

Phase 1 inventory found no sink that consumes attacker-influenced data. The file has two functions:

- `start_link/0` (line 9) — calls `supervisor:start_link({local, ?MODULE}, ?MODULE, ok)` with three compile-time-constant arguments. No parameters, so no data path.
- `init(ok)` (line 12) — pattern-matches the static atom `ok` and returns a literal child specification `{{one_for_one, 3, 10}, Workers}`, where `Workers` names `elixir_config` and `elixir_code_server` with static MFA tuples (`start_link`, `[]`), `permanent` restart, `2000` shutdown, `worker` type.

Every value in this file is a source-level literal. There is no `binary_to_term`, `list_to_atom`/`binary_to_atom`, `open_port`/`os:cmd`, file/path/archive operation, network call, crypto primitive, NIF boundary, ETS/match-spec, or deserialisation. The only "callable references" are the MFA tuples the supervisor will invoke, but they are hardcoded module/function atoms with empty argument lists — not derived from any input. The only guard is the `init(ok)` clause head, and its argument is the constant `ok` fixed in `start_link/0`, so it is not an attacker-reachable validation check. None of the folder's Unicode/identifier obligations (bidi, NFC, mixed-script, escapes, UTF-8) apply — this is supervision code, explicitly out of that scope.

## Risks / caveats
- The dismissals below rest on one external fact I verified by reading the caller: `elixir_sup:start_link/0` is invoked only from `elixir:start/2` (`elixir.erl:107`) with no arguments, at application boot by the OTP application controller. If a future change added a parameter to `start_link/0` or threaded caller-supplied config into the child spec, the "no external data path" basis would need re-checking — but as written there is no such parameter.
- Design-level observations that are **not** defects and are intended behaviour (documented in the folder threat model): a crash of `elixir_code_server` aborts all in-flight compilation in the node, and the `{one_for_one, 3, 10}` intensity terminates the supervisor if a worker restarts >3 times in 10s. Both are standard OTP semantics and require no attacker capability; neither is a security finding.
- No Erlang execution was needed: there is no behavioural claim (no decode, guard, loop, or computation) to reproduce — the file is a static configuration returned to the OTP framework.

### Ruled out

- **`supervisor:start_link/3` (line 10) — process/name registration & child startup (Shared mutable state / Code execution)**\
  Reason — all three arguments (`{local, elixir_sup}`, `elixir_sup`, `ok`) are compile-time literals; the registered name and the MFAs the supervisor will `apply` are fixed, with no input flowing from the sole caller `elixir:start/2` (which passes nothing).\
  Axes examined: argument origin (caller `elixir.erl:107` passes zero args), name value, MFA values, init-arg value | Not examined: none\
  Basis — verified

- **`init/1` child specification (lines 12–35) — callable references the supervisor invokes (Code execution)**\
  Reason — the child spec is a literal; module names (`elixir_config`, `elixir_code_server`), functions (`start_link`), and args (`[]`) are constants, so no external value can redirect what is started or with what arguments.\
  Axes examined: module atom, function atom, arg list, restart/shutdown/type fields, all literal | Not examined: none\
  Basis — verified

- **`init(ok)` clause head (line 12) — validation predicate / defensive guard**\
  Reason — not a defence against attacker input; the argument is the constant `ok` hardcoded at `start_link/0` line 10, so the clause is always reachable with the matching value and can never be driven by external data. It is not a dead defensive check either (it matches on every real invocation).\
  Axes examined: init-arg origin and value (constant `ok`) | Not examined: none\
  Basis — verified

- **Deserialisation / `binary_to_term` / atom-creation sinks**\
  Reason — none present; grep of the file shows no `binary_to_term`, `list_to_atom`, `binary_to_atom`, `file:consult`, or any decode path.\
  Axes examined: presence of each named unsafe call in the file body | Not examined: none\
  Basis — verified

- **File / path / archive / network / crypto / NIF / ETS-match-spec sinks**\
  Reason — none present; the file performs no I/O, no filesystem or path construction, no outbound request, no cryptographic operation, no NIF load, and no ETS match specification. It only returns a term to the OTP framework.\
  Axes examined: full file read (36 lines), enumerated each sink class against the source | Not examined: none\
  Basis — verified

### Did I verify this?

Every ruled-out entry is `verified`, and each `Not examined` list reads `none`. Substantiation:

- I read the entire file (`read elixir_sup.erl:1–36`) — it is 36 lines, so the search space is the whole file, fully covered, not a partial search.
- I traced the only inbound edge: `grep` for `elixir_sup` across the folder returned exactly one call site, and I read it (`read elixir.erl:95–113`). `start_link/0` takes no arguments and passes the literal `ok`; therefore no external data can reach any sink in this file. This is the one cross-file dependency, and it is confirmed rather than assumed, so no `Depends on:` line is required.
- The "sinks" here are all constant literals, so there are no datatype value-axes (path/identifier/integer/header) to disagree over — the axis lists are `none` because the values are fixed at compile time, not because I declined to enumerate them.
- No `asserted` dismissals remain, so there is nothing to hand to the cross-file pass.

