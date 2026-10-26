# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/assertions.ex
<!-- sha256: 26b77f593b2b33c3c7d82737f274b0d77e7ee685210196302668f076470fe521 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/assertions.ex
<!-- sha256: 26b77f593b2b33c3c7d82737f274b0d77e7ee685210196302668f076470fe521 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

Two total-function / contract defects, both now verified by execution:

1. `catch_throw`/`catch_exit`/`catch_error` crash at compile time with an internal `FunctionClauseError` when handed a bracket-list-literal expression (unchanged from first pass, still `verified`).
2. **Upgraded from hedged assumption in first pass:** `trace/3`'s documented `{:receive, match_specification}` flag raises `ArgumentError` ("invalid match specification") at runtime, because `trace/3` wraps the caller-supplied match specification in an extra list before passing it to `:trace.recv/3`. The first pass left this path `asserted` ("no defect was apparent"); executing it revealed the defect.

## Bugfixes

#### `CSCAN-assertions-do-catch-list-expr-crash`

- **Finding ID** — `CSCAN-assertions-do-catch-list-expr-crash`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/ex_unit/assertions.ex:982` (`do_catch/2`), interacting with `mark_as_generated/1` at lines 754-759.
- **Class** — Missing clause / non-total function (list-guard clause shadows the fallback for arbitrary expressions).
- **Impact** — A caller who writes `catch_throw([1, 2])` (or `catch_exit`/`catch_error` with any list-literal expression whose elements are not AST `{name, meta, context}` triples) gets a compile-time `FunctionClauseError` deep inside `ExUnit.Assertions.mark_as_generated/1` instead of the documented runtime assertion behaviour ("Expected to catch throw, got nothing"). The crash aborts compilation of the whole test module and its message points at ExUnit internals, telling the caller nothing about what to fix.
- **Evidence** — `verified`.
  Command: `elixir /tmp/t2.exs` with `test "..." do catch_throw([1, 2, 3]) end`.
  Output:
  ```
  ** (FunctionClauseError) no function clause matching in anonymous fn/1 in ExUnit.Assertions.mark_as_generated/1
      # 1
      1
      (ex_unit 1.21.0-dev) lib/ex_unit/assertions.ex:755: anonymous fn/1 in ExUnit.Assertions.mark_as_generated/1
      (elixir 1.21.0-dev) lib/enum.ex:1725: Enum."-map/2-lists^map/1-1-"/2
      (ex_unit 1.21.0-dev) lib/ex_unit/assertions.ex:982: ExUnit.Assertions.do_catch/2
      (ex_unit 1.21.0-dev) expanding macro: ExUnit.Assertions.catch_throw/1
  ```
  Root cause: at line 982 the user expression is passed to `mark_as_generated/1`. A bracket list literal such as `[1, 2, 3]` is a plain list AST, so it matches the `mark_as_generated(vars) when is_list(vars)` clause (line 754), which `Enum.map`s expecting each element to be a `{name, meta, context}` variable triple; the integer `1` matches no clause of the inner `fn`. The intended `mark_as_generated(other) -> other` fallback (line 759) never gets a chance because the `is_list` clause is tried first. (Sigil charlists like `~c"boom"` are safe — they remain a `{:sigil_c, …}` 3-tuple in the AST — so only `[...]`-syntax lists trigger it.)
- **Suggested fix** — Do not send an arbitrary expression through the variable-list helper. The smallest change is to stop wrapping it: at line 982 use `_ = unquote(expr)` (the `_ =` already suppresses unused-value warnings). If marking the node generated is genuinely wanted, guard the `is_list` clause so it only fires for lists of variable triples, or add an explicit `mark_as_generated(list)` clause that recurses element-wise safely — but for `do_catch` the expression should simply be left untouched.

#### `CSCAN-assertions-trace-receive-spec-double-wrap`

**Upgraded from hedged assumption in first pass.**

- **Finding ID** — `CSCAN-assertions-trace-receive-spec-double-wrap`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/ex_unit/assertions.ex:1132-1149` (`trace/3`); specifically the `{:receive, spec}` accumulation at line 1134 (`{[spec | receive], call, process}`) feeding `:trace.recv(session, receive_specs, [])` at line 1149.
- **Class** — Documented-contract violation (implementation wraps the caller's value in one list level too many before crossing the `:trace.recv/3` boundary).
- **Impact** — The `@doc` for `trace/3` (lines 1092-1093) documents the flag `{:receive, match_specification}` — "trace only received messages matching the given match specification. See `:trace.recv/3`" — and `:trace.recv/3`'s second argument is itself a match specification, i.e. a list of `{MatchHead, Guard, Body}` clauses. A caller following the documentation passes such a spec (e.g. `{:receive, [{:_, [], []}]}`). `trace/3` accumulates that value into `receive_specs` and hands the accumulator list to `:trace.recv/3`, producing `[[{:_, [], []}]]` — a list *containing* a match spec, which `:trace.recv/3` rejects with `ArgumentError, "invalid match specification"`. Only the undocumented single-clause form `{:receive, {:_, [], []}}` (a bare `{head, guard, body}` tuple, not a match specification) happens to work, because the accumulator then becomes a valid one-clause spec. So the documented usage is unusable and the working usage is undocumented. New feature (`@doc since: "1.21.0"`), niche flag → Low severity.
- **Evidence** — `verified` (executed; `/tmp/trace_spec.exs`, `/tmp/recv_sanity.exs`).
  - Documented form crashes:
    ```
    1) test receive with full match_specification (per docs) (TraceSpecTest)
    ** (ArgumentError) errors were found at the given arguments:
       * 2nd argument: invalid match specification
    stacktrace:
      (kernel 11.0.2) trace.erl:1510: :trace.recv({#Reference<...>, {:ex_unit_trace, 0}}, [[{:_, [], []}]], [])
      trace_spec.exs:19: (test)
    ```
    (input flag: `{:receive, [{:_, [], []}]}` → `:trace.recv` received `[[{:_, [], []}]]`.)
  - Undocumented single-clause form passes: `{:receive, {:_, [], []}}` runs green in the same file.
  - Boundary sanity, calling `:trace.recv/3` directly:
    - `:trace.recv(s, [{:_, [], []}], [])` → returns `1` (a full match spec is accepted directly — this is exactly what a user passes as the flag value).
    - `:trace.recv(s2, [[{:_, [], []}]], [])` → raises `%ArgumentError{message: "...invalid match specification..."}` (the extra list level `trace/3` adds is what breaks it).
- **Suggested fix** — Reconcile code and docs. Either (a) treat each `{:receive, spec}` value as a full match specification and pass a single one straight through — `:trace.recv(session, spec, [])` — rejecting/merging multiples explicitly; or (b) keep the clause-accumulation model but document the flag as `{:receive, match_spec_clause}` where the value is a single `{head, guard, body}` tuple, and update the example accordingly. As written, the `@doc`'s "match specification / see `:trace.recv/3`" wording and the implementation disagree.

## Risks / caveats

- Finding 1's trigger requires a bracket-list *literal* as the caught expression (`catch_throw([...])`); the common form — a call that throws/exits/errors — is a 3-tuple AST and is unaffected (Low).
- Finding 2 affects only the `{:receive, spec}` variant of `trace/3`. The plain `:receive`, `:send`, and `{:call, {M,F,A}}` flags, the unknown-flag `ArgumentError`, and the `try/after` session teardown all execute correctly (verified below), so only the match-spec sub-path is broken.
- I did not construct a genuinely *multi-clause* documented match spec to see whether the two-flag accumulation collides with a single full-spec flag; the single-flag documented path already fails, which is sufficient to establish the defect.

### Ruled out

These entries are re-affirmed from the first pass. They do not carry `Axes examined / Not examined` fields (none was assigned), so each remains a reasoned dismissal at the stated confidence; none was upgraded because re-execution (where applicable) reproduced the benign behaviour.

- **`trace/3` plain-flag, call-flag, send-flag, and unknown-flag paths (lines 1130-1157)** — Confidence: high. `verified` by execution (`/tmp/trace_test.exs`): `[:receive]`, `[:send]`, `[call: {Map, :get, 2}]` each deliver the documented trace tuples and pass their `assert_receive`; `trace/3` returns the value of `fun` (asserted `:ret_value`); an unknown flag raises `ArgumentError, "unknown trace flag: :bogus"` exactly as coded at line 1137; `:trace.session_destroy/1` runs in the `after` block. No defect.
- **`assert_raise/3` `cond` with no `true` branch (lines 831-835)** — Confidence: high. A `CondClauseError` results only if `message` is neither a `String` nor a `Regex`, which the `@doc` explicitly forbids ("which can be a `Regex` or an exact `String`"). Contract misuse, out of scope.
- **`assert_raise/2` not catching `throw`/`exit` (line 862, `rescue` only)** — Confidence: high. Documented to assert an *exception* is raised; `throw`/`exit` are not exceptions, matching the stated contract. Not a defect.
- **`extract_args/2` uses `String.to_unsafe_atom/1` (line 364)** — Confidence: high. `verified`: the module compiles and an `assert some_fun(1,2) == 3` test (which forces `extract_args` at compile time) compiled and ran cleanly (`/tmp/t3.exs`), so the function exists in this source tree.
- **`ExUnit.AssertionError.exception/1` `with` "inverted" logic (lines 46-53)** — Confidence: high. Traced: the body raises exactly when `:message` is present and non-binary, and falls through (building the struct) for a binary message or absent key. Correct.
- **`assert_in_delta` replacing vs `refute_in_delta` appending the custom message (lines 922-925 vs 1183-1191)** — Confidence: high. Asymmetric but each matches its own documentation/behaviour; inclusive/exclusive boundary semantics `verified` (`/tmp/t.exs`). Not a bug.

### Did I verify this?

- **Finding 1 (do_catch list crash)** — Yes, `verified` by execution; unchanged and reproduced conceptually from the first pass's captured stack trace. Confidence high.
- **Finding 2 (trace/3 `{:receive, spec}`)** — Yes, `verified` by execution this pass. I ran both the documented full-match-spec form (crashes) and the undocumented single-clause form (passes), and confirmed at the `:trace.recv/3` boundary directly that a bare match spec is accepted while the extra-wrapped list is rejected. This is the hedge the first pass flagged (`trace/3` was `asserted`, "no defect apparent"); it is now verified as a defect, not cleared. Confidence high.
- **Ruled-out `trace/3` non-spec flags** — Upgraded from `asserted` to `verified` this pass by running each flag path; behaviour matches the docs. No axes remain untried for these specific flag paths.
- **Other ruled-out entries** — Re-affirmed; those marked `verified` in the first pass were established by execution and are unchanged.

### Dropped in re-review

None. Both first-pass conclusions are preserved; the previously-`asserted` `trace/3` caveat is upgraded to a verified finding rather than dropped.
