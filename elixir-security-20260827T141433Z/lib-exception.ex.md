# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/exception.ex
<!-- sha256: 021b8f94b83c8f598086e602a75f14d8680091a6e451cf5da2401b85359009bf -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/exception.ex
<!-- sha256: 021b8f94b83c8f598086e602a75f14d8680091a6e451cf5da2401b85359009bf -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28T (hedge-verification pass on Elixir 1.21.0-dev / OTP 29, built from source under review)_

Re-review of the first-pass report. Two hedges were driven with executed evidence: (1) the `error_info` `apply` dismissal — an axis was tried and the assertion turned out to be **wrong**, so it is upgraded to a low-confidence finding; (2) the ReDoS dismissal — the backtracking axis was executed and the dismissal is upgraded from `asserted` to `verified`. The `:elixir_errors.format_snippet` collaborator caveat was closed by reading the neighbour and confirming the arguments this file passes (empty Diagnostic, no `span`) force the bounded branch.

---

# Security Review — `lib/elixir/lib/exception.ex`

## 1. Summary

`Exception` and its structs perform formatting/inspection of error data. Re-examination with executed inputs found **one low-confidence finding**: `Exception.normalize/3` (and `blame/3` via it) will `apply/3` an **attacker-chosen module/function** if the caller passes an untrusted *stacktrace list* carrying a crafted `:error_info` map — a weaker precondition than the first pass asserted. The `:erl_eval` guard/match paths, the snippet-formatter DoS, and the `List.to_unsafe_atom` atom-growth path remain empirically confined.

## 2. Security

### Finding 1 — `apply(error_module, error_fun, …)` invokes an attacker-chosen function when `normalize/3`/`blame/3` is given an untrusted stacktrace (line 2690)

**Confidence: low.** **Upgraded from hedged assumption in first pass** (was "Ruled out / asserted").

`error_info/3` (line 2679) reads `opts[:error_info]` from the **first stacktrace entry**, then:

```elixir
error_module = Map.get(error_info, :module, module)     # 2682
error_fun    = Map.get(error_info, :function, :format_error)  # 2683
apply(error_module, error_fun, [erl_exception, [head | tail]])  # 2690
```

The first pass ruled this out as `asserted`, reasoning that "any actor able to attach a custom `error_info` is already executing `:erlang.error/3`." **Executed evidence refutes that precondition.** `Exception.normalize/3` is a documented public function that accepts an *arbitrary* stacktrace argument. Feeding it a hand-built list reaches the `apply`:

```elixir
st = [{:erlang, :whatever, [1], [error_info: %{module: Evil, function: :format_error}]}]
Exception.normalize(:error, :badarg, st)
# >>> Evil.format_error INVOKED via crafted error_info
# %ArgumentError{message: "errors were found at the given arguments: pwned"}
```

`Evil.format_error/2` ran — no `:erlang.error/3` was needed, only a crafted **term** passed to the public API. The gadget is bounded (a 2-arity exported function called with `[exception, stacktrace]`; wrapped in `try/rescue`, so a raise is swallowed and side effects still occur), and reachable via the `:badarg` (2562), `:system_limit` (2576) and generic-`other` (2661) normalize clauses.

- **Why low and not higher:** the realistic trigger is a caller passing an *untrusted* stacktrace — unusual, since stacktraces are normally `__STACKTRACE__` (runtime-generated, trusted). It becomes real only for tools that **reconstruct/deserialize stacktrace terms from untrusted sources** (error-tracking ingestion, distributed error reports, log replay) and then call `Exception.normalize`/`blame` on them. Under that usage an attacker selects which exported `M.f(exception, stacktrace)` runs.

- **Failure scenario:** a service deserializes an error report `{:error, :badarg, stacktrace}` from an untrusted peer and calls `Exception.normalize(:error, :badarg, stacktrace)` to render it; the peer sets `error_info: %{module: <any loadable module>, function: <any 2-arity export>}` and causes that function to be invoked with peer-influenced arguments.

- **Axis tried (refuted):** argument-value steering — `String.to_integer("x" | "999zzz" | "hi")` all produced `error_info.module == :erl_erts_errors`, fixed regardless of the argument. So the *normal* BIF path is not steerable by argument values; the risk is confined to the crafted-stacktrace path.

### Ruled out

- **`blame_mfa/3` → `:erl_eval.expr` (lines 353–411) — arbitrary-code-execution via arguments.** **Basis: verified.** Arguments reach only `:erl_eval.expr({:match, …}, binding, :none)` (line 365, pattern-match binding, not evaluation); the guard evaluation at line 407 (`:erl_eval.expr(erl_guard, …)`) runs an expression translated from the **module's own debug_info clause guards** (trusted bytecode via `:elixir_erl_pass.translate`), with argument values present only as data bound to `:VAR`. First-pass reproductions: a fun passed as an argument was never invoked ("OK no side effect: args are matched"); bad/missing modules returned `:error`; a real stdlib clause-eval (`Keyword.fetch!`) returned `{:ok, …}`. Supplying malicious *guards* requires supplying malicious compiled bytecode = pre-existing code execution. *Axes examined: fun-as-argument invocation; guard-clause evaluation on a real beam. Not examined: none material — args cannot reach an eval position, only a match/binding position.*

- **Parser-error snippet DoS — `format_snippet/7`, `n_spaces` (lines 853–1031) and collaborator `:elixir_errors.format_snippet/6`.** **Basis: verified** (empirical + neighbour read). First pass drove adversarial source end-to-end (5 000-char lines, `(`×200 nesting, >5-line spans, tabs, unterminated strings) through `Code.string_to_quoted` → `SyntaxError`/`TokenMissingError`/`MismatchedDelimiterError` `message/1`; output was strictly proportional to input, no amplification, no crash. **Collaborator caveat now closed:** reading `elixir_errors.erl:122-206` confirms the only unbounded-duplication sinks are `lists:duplicate(Length, …)` (line 193, `Length = calculate_span_length`) and `n_spaces(Column-1)` (line 190). exception.ex passes `%{}` as the Diagnostic at lines 1254/1267/1366, so `calculate_span_length/2` hits the `#{}` clause (line 164) → `Length = 1`; `Column` is the parser's 1-based error column, bounded by line length. `read_snippet`/`get_file_line` (the file-opening path, line 222) is *not* reachable from exception.ex — this file always passes the snippet inline, so there is no filesystem surface. *Axes examined: line length, line count, nesting depth, span length, column magnitude, tabs. Not examined: none.*

- **`List.to_unsafe_atom/1` atom-table exhaustion — `load_module/1` (line 1810).** **Basis: verified.** Atoms are created only from `:code.all_available()` names (finite on-disk beams) filtered to those matching the missing module; the attacker's module name is only `Atom.to_string`'d, never re-atomised. First-pass repro: 2 000 distinct bogus names created ≤74 atoms; 2 000 suffix-matching names created 0. Growth bounded by on-disk modules, not by distinct input. *Axes examined: distinct attacker names, suffix-matching names. Not examined: none.*

- **ReDoS — `String.replace(inspected, ~r/^(?=.+)/m, …)` in `Protocol.UndefinedError.message/1` (line 2105).** **Basis: verified** (upgraded from `asserted`; backtracking axis executed). Timed on hostile input: 1 000 000-char single line = 2.3 ms; 200 000 lines = 483 ms (linear in line count, one zero-width match per line); "many newlines then content" adversarial case = 2.3 ms (no catastrophic backtracking). The pattern is line-anchored with a zero-width lookahead, and its input is `inspect(value, pretty: true)`, bounded by inspect's default element limit. *Axes examined: single-line length, line count, newline-heavy backtracking bait. Not examined: none.*

- **`Code.ensure_loaded/1` module-loading side effect (line 1811) during `UndefinedFunctionError` formatting.** **Basis: asserted.** Only modules already on the trusted code path (matched from `:code.all_available()`) are loaded; not an untrusted-input trust boundary. *Axes examined: none executed. Not examined: whether an `@on_load` on a code-path module has observable side effects when a candidate is loaded during error formatting — not tried.*

- **`inspect/1` of secret-bearing terms (`format_banner`, `KeyError`, `Protocol.UndefinedError`, etc.).** **Basis: asserted.** Leaking struct contents into messages is inherent to exception display; the mitigation (`@derive {Inspect, except: […]}`) belongs to the struct holding the secret, not this file. *Axes examined: none. Not examined: whether any struct formatted here defeats a caller's `@derive` redaction — not tried.*

- **`did_you_mean` Jaro-distance cost (`KeyError` 1873, `UndefinedFunctionError` 2222).** **Basis: asserted.** Cost proportional to in-memory data (map keys / module exports), only on the blame/display path, no input amplification. *Axes examined: none executed. Not examined: worst-case exports-list size vs. key-count scaling — not benchmarked.*

- **`Macro.inspect_atom/2` (756–761, 1901) and `Code.Identifier.binary_op/1` (2003), `extract_anonymous_fun_parent/1` (753).** **Basis: asserted (collaborator).** These format already-trusted atoms (module/function/operator names sourced from stacktrace entries and `defexception` metadata) into strings; not driven with untrusted atoms here. *Axes examined: none executed. Not examined: neighbour-internal behaviour on a hostile atom — but the atoms reaching them originate from trusted stacktrace/module metadata, not source text.*

- **No eval / command / deserialization / filesystem-mutation sinks.** **Basis: verified** (Grep over the whole file): no `Code.eval*`/`compile*`, `System.cmd/shell`, `:os.cmd`, or `:erlang.binary_to_term` in executable code; all such tokens are docstring-only. `File.*`/`Path` references are exception *structs* whose `message/1` only formats (`:file.format_error/1`, `inspect(path)`, `Path.relative_to_cwd/1` — pure string op); no file operations, no path-traversal/TOCTOU surface.

## 3. Risks / caveats

- **Finding 1 hinges on caller trust of the stacktrace argument.** I demonstrated the `apply` fires from the public `normalize/3` with a hand-built stacktrace; I did **not** find an in-tree caller that feeds it an untrusted stacktrace. The residual risk is a usage-pattern risk (deserialized/relayed stacktraces), hence low confidence — but the mechanism is confirmed by execution, not assumed.
- **Still asserted (not executed):** the `Code.ensure_loaded` `@on_load` axis, the `inspect` secret-leak `@derive`-defeat axis, and the `did_you_mean` worst-case cost axis. These stay `asserted`; a later pass may execute them. I did not upgrade them because I have no executed evidence in either direction.
- **`blame_mfa` cost** is O(beam size) by its own docstring; I verified no *code-execution* escalation from arguments, not the absence of a large-beam CPU cost — but the beam read is a local trusted module, not untrusted input.
- Reproductions ran on Elixir 1.21.0-dev / OTP 29 built from the source under review (1-based parser columns, this build's `:code.all_available` behaviour). Incidental confirmation: a bidi char (`U+202E`) placed in a test `.exs` string was rejected by the tokenizer with "invalid bidirectional formatting character" — consistent with BIDI_CONTROL_REJECTED, though that guard lives in `elixir_tokenizer`, not this file.

## 4. Did I verify this?

- **Finding 1 (error_info `apply`):** **Yes, executed.** Ran `Exception.normalize(:error, :badarg, crafted_stacktrace)` and observed `Evil.format_error/2` invoked; ran `String.to_integer/1` with three inputs and observed the `error_info.module` fixed at `:erl_erts_errors` (argument-steering axis refuted). Both axes tried.
- **ReDoS dismissal:** **Yes, executed.** Timed the exact regex on 1M-char, 200k-line, and backtracking-bait inputs; all linear.
- **Snippet-DoS collaborator:** **Yes** — read `elixir_errors.erl:107-226` and confirmed the branch exception.ex's arguments select is bounded (span length forced to 1; no file open reachable).
- **blame_mfa, List.to_unsafe_atom, no-dangerous-sinks:** carried forward as **verified** from the first pass (executed reproductions + Grep), re-confirmed against the current source lines.
- **ensure_loaded `@on_load`, inspect-secret, did_you_mean cost, Macro/Code.Identifier collaborators:** **No executed evidence** — remain `asserted` with their untried axes named above.

## 5. Dropped in re-review

None. Every prior ruled-out entry is preserved; one (`error_info` `apply`) was **upgraded** to a finding rather than dropped, and one (ReDoS) had its basis strengthened `asserted → verified`. No prior finding existed to drop (the first pass reported none).

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`did_you_mean` Jaro-distance cost (`KeyError`, `UndefinedFunctionError`, lines 1873/2222).** **Basis: asserted.** Cost is proportional to already-in-memory data (map keys / module exports), incurred only on the blame/display path, with no input amplification.
