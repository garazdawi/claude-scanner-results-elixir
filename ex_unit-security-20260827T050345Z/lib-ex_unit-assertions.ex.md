# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/assertions.ex
<!-- sha256: 26b77f593b2b33c3c7d82737f274b0d77e7ee685210196302668f076470fe521 -->

_Analysed on 2026-08-27T05:11:08Z_

Both confirmed: `trace/3` destroys its session in the `after` clause even when `fun` raises (session count `{1,1}` — balanced, no leak), and `assert_raise/3` with an out-of-contract message type merely raises `CondClauseError` (a contract violation, no security impact). I have enough to conclude.

## Summary

`ExUnit.Assertions` is the compile-time assertion macro/function library imported into every ExUnit test case (`assert`/`refute`, `assert_receive`/`assert_raise`, `catch_*`, `trace/3`, `flunk`, etc.); it is developer-facing test tooling with no parser, decoder, deserializer, eval, command, path, network, or crypto sink, and I found no security defect reachable with untrusted data through its documented public API.

## Security

No findings.

The file is almost entirely macros that expand at compile time over the developer's own trusted test AST, plus runtime helpers whose only "output" of runtime values is `inspect/1`/`Macro.to_string/1` string-building into exception messages (inert strings handed to the formatter — no secondary interpreter, no format-string sink, no injection within this file). A full grep for the dangerous-sink families (`Code.eval*`, `binary_to_term`, `System`/`:os.cmd`, `Path`/`File`, `Module.concat`, `apply`, `Regex.compile`, `EEx`, `to_atom`) returned exactly one hit — the atom-creation call below — which I traced and ruled benign.

### Ruled out

- **`String.to_unsafe_atom("arg#{i}")` in `extract_args/2` (line 364) — atom creation.** Runs at *compile time* during macro expansion of the developer's own `assert`/`refute` source, not at runtime. The atoms created are always of the fixed form `argN` and are bounded by the arity of the call literally written in the source; across all compilations they are the same reused atoms (`verified`: `Macro.expand_once(quote(do: assert some_fun(a,b,c)))` expands to `{arg1, arg2, arg3} = {a, b, c}`, and `String.to_unsafe_atom("arg1") == :arg1` is `true`). No runtime/external value influences the count or names, so there is no atom-table-exhaustion path from untrusted data. `to_unsafe_atom` is deliberate here (an `argN` may not pre-exist, so `to_existing_atom` would fail); if hardening were ever desired the bounded set could be pre-interned, but there is no vulnerability to fix.
- **`assert_raise/3` `cond` without a catch-all (lines 832–835).** A `message` that is neither a binary nor a `Regex` raises `CondClauseError` (`verified`). This requires the caller to violate the documented `String | Regex` contract and has no security impact (a messier failure in the developer's own test), so it is out of scope.
- **`trace/3` session cleanup (lines 1140–1156).** `:trace.session_destroy/1` runs in the `after` clause and fires even when `fun` raises (`verified`: session count unchanged, `{1, 1}`), so a failing traced function does not leak a global trace session into sibling tests. Balanced.
- **`assert_receive` timeout path / `format_mailbox` (lines 606–657).** Walks the *test's own* mailbox; the displayed set is capped at `@max_mailbox_length` (10) and rendered via `inspect` (default limit), so there is no external-attacker-controlled unbounded work — any cost is self-inflicted by the test.
- **`translate_operator`, `__timeout__/2` guard, `expand_pattern` quote/`@` clauses.** All branches are reachable via documented usage; none is dead defensive code. `expand_pattern`'s `Macro.expand`/`Module.get_attribute` operate only on the developer's compile-time pattern AST and attribute names.

## Risks / caveats

- **Robustness-only asymmetry (not security):** the 2-arg `assert_raise/2` wraps `Exception.message(error)` in `check_error_message/2` (a `catch`), but the 3-arg `assert_raise/3` (lines 833–840) calls `Exception.message(error)` without that guard. A code-under-test exception whose `message/1` callback crashes would surface as that crash rather than a clean assertion failure. This is a developer/dependency bug with no trust-boundary crossing or security impact; flagged only so a follow-up is aware of the inconsistency.
- I treated the ExUnit threat model's premise as given: the test author (and thus the test AST reaching every macro) is trusted, and runtime values reaching assertion helpers are only ever `inspect`ed/compared, never fed to a downstream interpreter within this file. If a caller were to route this module's exception *messages* into a sink that re-interprets them (a shell, a query, EEx), that would be a defect in that caller, not here.
- `trace/3` (since 1.21.0) forwards developer-supplied match specifications and MFAs to Erlang's `:trace`; these are test-author-controlled, so I did not treat them as an untrusted boundary. If a project ever exposed `trace/3` flags to external input that assumption would need revisiting.

